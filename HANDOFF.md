# Handoff — 2026-05-22

**Head commit (project):** 722831a — ci: add casehub-eidos to dashboard, full-stack, and incremental builds
**Head commit (workspace):** 3fe6ee8 — docs: add blog entry 2026-05-22-no-end-users

## What Changed This Session

Rewrote the casehub dev workflow prompt snippet — added no-end-users context,
explicit workaround interrupt, continuous PLATFORM.md + protocol alignment during
design, and mid-session skip instruction. Dropped stale epic hygiene item (work-end
covers it). Formalised the no-workarounds rule as protocol PP-20260522-3b1ccd.

Fixed cross-repo journal routing bug in work-start/work-end: DESIGN_REPO was
set from workspace routing config when tracking a cross-repo issue, pointing
the journal at the wrong DESIGN.md. Layer 0 added to routing cascade; issue-repo
stored in .meta; work-end closes against the correct GitHub repo. Shipped to
cc-praxis main, pushed.

Verified and closed claudony#122 from handover. All repos pushed clean.

## Immediate Next Step

Start #20 — tutorial-strategy.md + all app deep-dives as field tutorials.
Run `work-start` referencing issue #20.

## What's Left

- `epic-atomic-human-task` — engine session, branch at engine#273 · S · Low
- 3 untracked plans in workspace (`plans/2026-05-19-*.md`) — pre-existing, not blocking

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #20 | tutorial-strategy.md + all app deep-dives as field tutorials | L | Low | Multi-file rewrite |
| #5 | Consolidate Connector + NotificationChannel into outbound SPI | M | High | No cross-repo touches until ready |
| #4 | Platform coherence audit — 32 cross-repo findings | L | High | Read-only analysis feasible now |
