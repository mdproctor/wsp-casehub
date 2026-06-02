# Handoff — 2026-06-02

**Head commit (project):** 19659a4 — chore: fix design routing — design-repo is project not workspace
**Head commit (workspace):** 0a60c8c — feat: promote blog entry from issue-141-doc-sync

---

## What Changed This Session (2026-06-02)

**Doc sync batch closed** — #141, #142, #143, #144:
- `casehub-eidos.md`: eval/ module, `delegation` field fix, eidos#23 status
- `docs/PLATFORM.md`: ChannelProjection capability row
- `casehub-qhorus.md`: Channel Read-model Projection subsection, Store SPIs, Module Structure
- `arc42stories-spec.md`: duplicate Writing Style removed, mode map fixed, `_universal.md` added to Generator pre-conditions
- `write-content/forms/technical-documentation.md`: stray `yes` typo fixed (synced to cc-praxis)
- Protocol PP-20260602-e36824: `external-api-surface-in-deep-dive` formalised

**CLAUDE.md:** design routing corrected — `design → project` (was `workspace`)

---

## Immediate Next Step

Discuss #3 (automate linked PR chain) — design is solid, key open question is SNAPSHOT timing strategy (poll vs retry). See conversation from this session for the analysis.

---

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Left

- `clinical` — recover stranded blog from `epic-3-multi-site-sub-case` · XS · Low
- CLAUDE.md mass update (ARC42STORIES.MD references, LAYER-LOG retirement) — cross-repo, each repo's own session
- Workspace branch cleanup — branches `issue-12` through `issue-19` scheduled for deletion 2026-06-04 (2 days)

---

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #3 | Automate linked PR chain across ecosystem repos | L | High | CROSS_REPO_PAT needed; SNAPSHOT timing open |
| #93 | Extract normative channel layout claudony → engine-api | M | Med | Blocked by claudony#142 |
| #106 | Update spec §7.1 oversight channel allowedTypes | XS | Med | Blocked by claudony#142 |
| #111 | Migrate actorId format to DID | XL | High | Needs ledger#108, ledger#110 |
| #7 | Platform foundation roadmap tracker | XL | High | Tracker — closes last |
| #13 | Cohesive Claude config design | XL | High | Needs dedicated design session |

---

## Key References

- Blog: `blog/2026-06-02-mdp02-two-writing-styles-one-commit.md`
- Protocol: PP-20260602-e36824 `external-api-surface-in-deep-dive` (garden)
- Branch cleanup due: `issue-12` through `issue-19` → 2026-06-04
