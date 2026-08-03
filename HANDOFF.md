# HANDOFF — casehub

**Date:** 2026-08-03
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

Decentralised repo deep-dives across all 28 CaseHub repos (#377) and rewrote the platform documentation index for consumer/contributor split (#401).

**#377 — Decentralise deep-dives:**
- Each repo now owns `docs/guides/consumer-guide.md` (app builders) + `docs/guides/contributor-guide.md` (platform builders)
- Parent aggregates via git subtree (same pattern as casehub-examples) — SHA provenance per repo
- All 28 repos source-audited by Opus agents against actual code, git history, GitHub issues — every repo had significant staleness (retired reactive tiers, phantom features, wrong counts, missing modules/SPIs, wrong class names)
- Sync infrastructure: `scripts/sync-guides.sh` (subtree split on `docs/guides/`), `scripts/guide-sync-config.json`, `.github/workflows/sync-guides.yml`
- Each repo's CLAUDE.md updated with `## Repo Guide` section pointing to local guides with maintenance instruction

**#401 — INDEX.md consumer/contributor split:**
- `docs/INDEX.md` is now the universal LLM entry point — routes to audience-specific indexes
- `docs/consumer-index.md` — capability map linking to all 28 repos' consumer guides with key types inline
- `docs/contributor-index.md` — architecture map with module counts and dependency chains
- 33 stale links fixed across 7 active docs, 29 old flat `docs/repos/casehub-*.md` files removed

## What's Left

- **Neocortex audit commit on wrong branch** — landed on `issue-198-expansion-drift-metrics` instead of main. Needs cherry-pick when that branch lands.
- **Ledger audit commit on feature branch** — landed on `feat/aml-115-content-sanitiser-rename` (cherry-picked to main already, but feature branch has a copy too)
- **#359 partially resolved** — automated audit tooling and API catalogue generation still open. Comment posted on the issue.
- **Prompt snippet update** — the corrected section 4 for the standard work prompt was provided in conversation but not persisted to a file. User should save it.

## References

- #377 closed — `097ad104` on main
- #401 closed — `274fc6e4` on main
- #359 updated with partial resolution comment
