# Handoff — 2026-05-29

**Head commit (project):** e417e4a — protocol(PP-20260529-ce2de0): engine-api-scope-rule
**Head commit (workspace):** 3387a88 — feat: promote blog entry from issue-90-91-94-doc-batch

---

## What Changed This Session (2026-05-29)

Issue list refreshed — 8 stale entries pruned (work#221, qhorus#200, work#229, engine#281, work#214, PR#370, engine#342/#331/#248 — all CLOSED). Doc batch #90/#91/#94 completed and closed: PLATFORM.md memory adapter rerouting (ADR-0008 amendment), casehub-clinical.md SPI sync, arc42stories layer notation and matrix. All committed, delivered to casehubio/parent upstream. Blog entry mdp02 written and published.

Hygiene note: `squash/wip-main-20260529-015357` project branch exists from an interrupted squash — needs review or cleanup. Garden entry GE-20260521-eaa1e1 revised (ff-only shortcut when branch base is still ancestor of extended main).

---

## Immediate Next Step

Stamp `issue-65-bom-and-doc-sync` workspace branch with close marker (parent#65 is CLOSED, branch has just the init commit). XS·Low, 2 minutes.

---

## Cross-Module

**Blocked by:**
- `work` — `work#225` (wire ExpiryLifecycleService to CaseSignalSink) gates `parent#64` · S · Low

---

## What's Left

- `flow#1` PR — merge CLAUDE.md platform awareness boilerplate · XS · Low
- `work#225` — wire ExpiryLifecycleService to CaseSignalSink · S · Low
- `parent#64` — PLATFORM.md signal bridge update (gate: work#225) · S · Low
- `new-repo-checklist.md` — missing AGENTIC-HARNESS-GUIDE step · XS · Low
- `issue-65-bom-and-doc-sync` workspace branch — needs close marker stamp · XS · Low
- `squash/wip-main-20260529-015357` project branch — stale squash branch, review/cleanup · XS · Low

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key References

- Blog: `blog/2026-05-29-mdp02-three-syncs-two-principles.md`
- Garden: GE-20260521-eaa1e1 revised — ff-only shortcut when branch base is still main ancestor
