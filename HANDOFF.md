# HANDOFF — casehub

**Date:** 2026-06-27
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**Docs batch (#314, #313, #210) + two new application repos (soc, fsitrading) + build infra cleanup.**

- Closed #314 (casehub-work deep-dive for WorkItemCreator SPI), #313 (agent-langchain4j rename), #210 (rag-api rows already present).
- Created `casehubio/soc` and `casehubio/fsitrading` — full Maven scaffolds, CLAUDE.md with domain-specific docs, workspaces, symlinks, forks, remotes. Both pushed and registered in PLATFORM.md + APPLICATIONS.md.
- Moved 5 modules (iot, desiredstate, ras, workers, ops) from `modules-local.csv` into CI pipeline. Deleted `modules-local.csv` — all modules must be in CI. Added soc + fsitrading to applications CSV. Updated both dashboards with all 8 missing repos.
- Updated `new-repo-checklist.md` with 6 missing items discovered during bootstrap (Project Artifacts, IntelliJ MCP, Work Tracking behaviours, workspace blog-routing, module publishing verification, CI mandate).
- Fixed IntelliJ MCP routing across parent, soc, fsitrading CLAUDE.md — `mcp__intellij__*` disabled (memory leak).

## Immediate Next Step

Pick next work from the backlog — #315 (neural-text deep-dive sync, XS/Low) is the only remaining non-design, non-epic issue scoped for this repo. Or start a new session on `soc` or `fsitrading` to begin domain research.

## What's Left

- `qhorus#309` — add `isActive()` to CommitmentState · XS · Low
- `ledger#159` — normalize remaining event producers to dual-channel · S · Low
- `#315` — docs: sync casehub-neural-text deep-dive for Matryoshka + quantization · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon; channel taxonomy patterns ready |
| — | casehub-soc first session | M | Med | Domain research + brainstorming |
| — | casehub-fsitrading first session | M | Med | Domain research + brainstorming |
