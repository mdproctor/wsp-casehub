# Handoff — 2026-05-27

**Head commit (project):** 8acb1c3 — chore(#78): register all ecosystem repos across CI, docs, and peer lists
**Head commit (workspace):** c2ef444 — blog entry what-wasnt-on-the-map

---

## What Changed This Session (2026-05-27)

Ecosystem registration audit. Seven repos (platform, eidos, openclaw, drafthouse, life, quarkmind, flow) were missing from at least one of five registries: CLAUDE.md peer list, full-stack CI, incremental CI, PLATFORM.md, APPLICATIONS.md/AGENTIC-HARNESS-GUIDE.

Fixed all gaps — see casehubio/parent#78. quarkmind added to full-stack-build.yml. flow registered in both CI builds with "tier TBD" marker in PLATFORM.md (platform coherence analysis deferred). GH_REPO map in full-stack CI switched to full `org/repo` paths to handle cross-org repos (quarkmind, flow are under mdproctor). Committed, squashed 11→6, pushed.

Opened mdproctor/flow#1 — CLAUDE.md platform awareness boilerplate (repository role, build commands, work tracking, dev workflow).

---

## Immediate Next Step

*Unchanged — `git show HEAD~1:HANDOFF.md`*

Top candidates (still unblocked from engine#349 + PR #370):
- `work#225` — wire ExpiryLifecycleService to CaseSignalSink · S · Low
- `work#221` — add SIGNAL_RECEIVED to WorkEventType · S · Low
- `qhorus#200` — WatchdogAlertEvent + ConnectorAlertBridge · S · Med

PR #370 (engine correctness batch) still needs review/merge.

---

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Left

- PR #370 needs review/merge · S · Low
- mdproctor/flow#1 needs merge (CLAUDE.md boilerplate) · XS · Low
- `work#229` — db/migration rename, coordinate with aml, clinical, devtown · M · Med
- `new-repo-checklist.md` missing AGENTIC-HARNESS-GUIDE step — noted, not fixed · XS · Low
- flow platform coherence analysis deferred (tier, overlaps, conventions) · M · High

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key References

- Engine PR: casehubio/engine#370 — correctness batch (#342, #331, #248, #338)
- Ecosystem registration: casehubio/parent#78 — full audit and fix
- flow boilerplate PR: mdproctor/flow#1
- Issue triage detail: `git show HEAD~1:HANDOFF.md`
