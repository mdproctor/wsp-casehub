# Handoff — 2026-05-26

**Head commit (project):** 3f99013 — docs: promote design spec from issue-369-engine-correctness-batch  
**Head commit (workspace):** current main

---

## What Changed This Session (2026-05-26 afternoon)

Engine correctness batch landed. Four S/Low/High bugs fixed in a single branch, PR #370 open:
- **#342** — CaseLedgerEntry.traceId always null (OTel context severed by @ObservesAsync)
- **#331** — CapabilityTarget PlanItem stuck RUNNING when Quartz retries exhausted
- **#248** — JpaSubCaseGroupRepository atomic counter race under PostgreSQL READ COMMITTED
- **#338** — WorkItemLifecycleAdapter collapsed REJECTED/EXPIRED → FAULTED; added PlanItemStatus.REJECTED

ADR-0002 filed (stage autocomplete triggers on any terminal PlanItem state). Engine branch closed, CLAUDE.md synced.

---

## Immediate Next Step

Pick any unblocked item. Top candidates now that engine S/Low correctness bugs are done:

**Now-unblocked (engine#349 ✅, engine#370 branch open):**
- `work#225` — wire ExpiryLifecycleService to CaseSignalSink · S · Low
- `work#221` — add SIGNAL_RECEIVED to WorkEventType · S · Low
- `qhorus#200` — WatchdogAlertEvent + ConnectorAlertBridge · S · Med

**M/Low/High — next highest value:**
- `engine#338` — fix: WorkItemLifecycleAdapter collapses REJECTED/EXPIRED/ESCALATED → FAULTED ✅ **DONE in #370**
- `engine#339` — ✅ already closed
- `clinical#26` — feat: engine_case_id on ProtocolDeviation and AdverseEvent · M · Low
- `engine#338` fixes in #370 also unblock design definitions relying on REJECTED routing

---

## Issue Triage — All Remaining Open Issues

### ── DOCS / CONFIG (no Java) ────────────────────────────────────────────

| Issue | Repo | Scale | Complexity | Impact | Blocked? |
|-------|------|-------|------------|--------|----------|
| parent#47 | parent | S | Low | Low | No — workspace CLAUDE.mds are symlinks |
| engine#333 | engine | S | Med | Med | No — inputSchema/outputSchema mini-DSL inconsistency |
| devtown#45 | devtown | S | Low | Low | No |
| parent#64 | parent | S | Low | Med | Partially unblocked — also needs work#225 + qhorus#200 |
| work#159 | work | M | Med | Med | No |
| parent#66 | parent | L | Med | Med | No |

---

### ── CODE (Java) — S scale ─────────────────────────────────────────────

**NOW UNBLOCKED (engine#349 ✅):**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| work#225 | work | feat: wire ExpiryLifecycleService to CaseSignalSink | NOW UNBLOCKED |
| work#221 | work | design: add SIGNAL_RECEIVED to WorkEventType | NOW UNBLOCKED |
| qhorus#200 | qhorus | feat: WatchdogAlertEvent + ConnectorAlertBridge | NOW UNBLOCKED |

**S / Low / Med — test additions and small features:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#281 | engine | test: JpaReactivePlanItemStore has no contract test | No |
| work#214 | work | feat: add @DefaultBean no-op RoutingCursorStore to work-core | No |
| work#211 | work | feat: WorkItemService.extend(id, newExpiresAt) for BreachDecision.Extend | No |
| work#206 | work | test: add positive-path outcome filter test | No |
| work#201 | work | test: clearTemplates() should also clear WorkItems and AuditEntries | No |
| work#192 | work | feat: audit CREATE_DENIED for excluded direct assignee | No |
| claudony#131 | claudony | perf: replace 500ms SSE tick with ChannelEventBus-driven push | No |

**S / Med / High — important design gaps:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| qhorus#199 | qhorus | design: TrustGateService injectable but never called | No |
| qhorus#197 | qhorus | design: EVENT telemetry in DB only — invisible in distributed traces | No |
| eidos#8 | eidos | design: EpistemicallyWeak should be soft preference signal | No |

**S — blocked:**

| Issue | Repo | What | Blocked by |
|-------|------|------|-----------|
| clinical#28 | clinical | fix: AeEscalationListener uses engine.internal.event | Engine must promote CaseLifecycleEvent to public SPI |

---

### ── CODE (Java) — M scale ─────────────────────────────────────────────

**M / Low / High — high impact, clear path:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#338 | engine | ✅ DONE in PR #370 | — |
| engine#339 | engine | ✅ already closed | — |
| clinical#26 | clinical | feat: engine_case_id on ProtocolDeviation and AdverseEvent | No |
| clinical#27 | clinical | feat: AdverseEvent.escalationStatus field | No |
| work#229 | work | chore: rename db/migration/ → db/work/migration/ | No — but blocks aml, clinical, devtown |

**M / Med / High:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#337 | engine | design: WorkOrchestrator CDI strategy resolution | No — unblocks #336 |
| engine#336 | engine | design: SelectionContext trust scores | Blocked on #337 |
| engine#256 | engine | Expose CaseContext factory on SPI | No |
| engine#274 | engine | feat(blackboard): hydrate BlackboardRegistry from PlanItemStore | No |
| engine#212 | engine | feat: casehub-work multi-instance WorkItem spawning | No |
| qhorus#166 | qhorus | feat: dispatch MessageObserver after transaction commit | No |
| claudony#116 | claudony | feat: enable reactive Qhorus stack for PostgreSQL | No |

**M / Low / Med:**

*Unchanged — `git show HEAD~1:HANDOFF.md`*

---

### ── CODE (Java) — L / XL scale ────────────────────────────────────────

| Issue | Repo | Scale | Complexity | Blocked? |
|-------|------|-------|------------|----------|
| connectors#4 | connectors | L | High | NOW UNBLOCKED (engine#349 ✅) |
| clinical#21 | clinical | L | Med | No |
| clinical#11 | clinical | L | Med | Blocked on connectors#2 |
| parent#56 | parent | L | High | No |
| parent#5 | parent | L | High | No |
| parent#3 | parent | L | High | No |
| parent#13 | parent | L | Med | No |
| qhorus#162 | qhorus | L | High | No |
| qhorus#132 | qhorus | L | Med | No |
| claudony#102 | claudony | L | Med | No |
| claudony#98 | claudony | L | Med | No |
| engine#340 | engine | L | High | No |

---

## Blocking Chain Summary

```
engine#349 ✅ + engine#370 (PR open) → unblocks:
  work#225, work#221, qhorus#200, connectors#4
  parent#64 (partially — also needs work#225 + qhorus#200)

engine#337 → unblocks engine#336
connectors#2 → unblocks clinical#11
work#229 (db/migration rename) → triggers aml, clinical, devtown config updates
clinical#28 needs: engine to promote CaseLifecycleEvent to public SPI
```

---

## Key References

- Engine PR: casehubio/engine#370 — correctness batch (#342, #331, #248, #338)
- ADR-0002: stage autocomplete triggers on any terminal PlanItem state
- Protocol PP-20260526-3a22d3: OTel traceId capture before fireAsync()
- work#229 filed: db/work/migration rename — coordinate with aml, clinical, devtown
