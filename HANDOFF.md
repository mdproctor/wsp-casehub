# Handoff — 2026-05-28

**Head commit (project):** 6fc667f — docs: sync platform deep-dives for recent implementation work
**Head commit (workspace):** c184f2a — docs: promote blog entry from issue-72-doc-sync-batch

---

## What Changed This Session (2026-05-28)

Documentation sync batch. Nine issues resolved (closed #78 + #72, #75, #76, #77, #79, #80, #81, #82 — all casehubio/parent). Changes: casehub-engine.md (signal bridge, AgentRoutingStrategy SPI, casehub-work-core routing dep removed), casehub-openclaw.md (Epics 2+3 — in-memory ChannelContextWindow, OpenClawHookClient, removed bogus datasource section), PLATFORM.md (ClaudonyChannelBackend uses SSE not WebSocket), casehub-life.md (Layer 1 complete), quarkmind.md (LAYER-LOG Layer 1+2 written), AGENTIC-HARNESS-GUIDE (domain entity discipline principle). Merged locally after GitHub showed spurious DIRTY merge state (GE-20260528-de4fc4 submitted to garden).

---

## Immediate Next Step

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## What's Left

- PR #370 needs review/merge · S · Low
- mdproctor/flow#1 needs merge (CLAUDE.md boilerplate) · XS · Low
- `work#225` — wire ExpiryLifecycleService to CaseSignalSink · S · Low
- `work#221` — add SIGNAL_RECEIVED to WorkEventType · S · Low
- `qhorus#200` — WatchdogAlertEvent + ConnectorAlertBridge · S · Med
- `work#229` — db/migration rename, coordinate with aml, clinical, devtown · M · Med
- `new-repo-checklist.md` missing AGENTIC-HARNESS-GUIDE step · XS · Low
- flow platform coherence analysis deferred · M · High
- `issue-65-bom-and-doc-sync` workspace branch — open, 3 days, no close marker · XS · Low

---

## What's Next

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

## Key References

- Blog: `blog/2026-05-28-mdp01-keeping-the-docs-honest.md`
- Garden: GE-20260528-de4fc4 — GitHub DIRTY merge state gotcha
- Issue triage detail: `git show HEAD~1:HANDOFF.md`
