# Handoff — 2026-05-25

**Head commit (project):** 367b14a — protocol(PP-20260525-6442ba,PP-20260525-724406)
**Head commit (workspace):** 3c461a5 — blog entry openclaw-governance-memory

## What Changed This Session

Large session in two halves. First half: closed 15 doc issues on parent (PLATFORM.md,
casehub-qhorus.md, casehub-ledger.md, casehub-clinical.md, casehub-devtown.md, three new
protocol files, new issues for signal bridge work). Second half: research conversation on
OpenClaw/Sai/Coworker.ai → 1700-line research spec → bootstrapped casehub-openclaw and
casehub-life repos in full (GitHub repos, Maven skeletons, CLAUDE.mds, LAYER-LOGs, CI,
workspaces, parent plumbing, website, 22-step checklist, fork + remote setup). Checklist
verified against existing repos — 3 dispatch chain errors found and corrected, 2 new
protocols captured.

## Immediate Next Step

Start issue #20 — tutorial-strategy.md + all app deep-dives as field tutorials.
Run `work-start` referencing issue #20.

## Cross-Module

**engine#350** — engine must add `openclaw` to its dispatch list (cannot commit from parent
session). Blocks casehub-openclaw from being triggered by upstream engine changes.

**aml#33** — aml build.yml missing `repository_dispatch` trigger. Filed; needs action in
aml session.

**clinical#36** — casehub-platform scope should be `runtime` not compile default in
runtime/pom.xml. Filed; low urgency (currently works).

## What's Left

- 3 untracked plans in workspace (`plans/2026-05-19-*.md`) — pre-existing, not blocking

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #20 | tutorial-strategy.md + app deep-dives as field tutorials | L | Med | Next planned work |
| — | casehub-openclaw Epic 2: hook API client | M | Med | Blocked until engine#350 dispatch fixed |
| — | casehub-life Epic 2: Layer 1 household domain model | M | Low | Ready to start |
| — | casehub-memory SPI in platform (platform#27) | L | High | Design needed before implementation |
