# Handoff — 2026-05-26

**Head commit (project):** 4db7c0f — docs(#73,#74,#65…): full batch merge to main
**Head commit (workspace):** current main

---

## What Changed This Session (2026-05-26)

Massive S/XS sweep across all repos. Two protocols written (PP-20260525-607b33 Flyway repo-scoped paths, PP-20260526-6d39e5 opaque cross-module identifiers). Workspace isolation fixed (life/, openclaw/ now own repos). All open branches merged to main except engine (PR #368 — merged by user at session end).

**Repos touched:** parent, engine, work, qhorus, claudony, aml, devtown, connectors, clinical, eidos, garden

**Issues closed this session:** parent#65–71, #73, #74; aml#16, #18, #20, #22, #23, #28, #33; devtown#23, #25, #26, #46; connectors#3; clinical#12; work#164, #198, #224, #223; qhorus#201; engine#243, #332, #350, #367; claudony#113, #129; eidos#2, #3, #9 (many already done/invalid)

**New issues filed:** work#229 (Flyway rename), engine#367 (blocking=true)

---

## Immediate Next Step

Pick any unblocked S/Low/High code item from the triage below. engine#342, engine#331, or engine#248 are the highest-value next actions — correctness bugs, clear path, no deps.

---

## Issue Triage — All Remaining Open Issues

### ── DOCS / CONFIG (no Java) ────────────────────────────────────────────

| Issue | Repo | Scale | Complexity | Impact | Blocked? |
|-------|------|-------|------------|--------|----------|
| parent#47 | parent | S | Low | Low | No — but requires edits in peer repo sessions (workspace CLAUDE.mds are symlinks) |
| engine#333 | engine | S | Med | Med | No — inputSchema/outputSchema mini-DSL inconsistency, docs only |
| devtown#45 | devtown | S | Low | Low | No — update gastown-casehub-analysis-v2.md status |
| parent#64 | parent | S | Low | Med | Partially unblocked — engine#349 ✅, but also needs work#225 + qhorus#200 |
| work#159 | work | M | Med | Med | No — normative layer alignment: map WorkItem lifecycle to Qhorus speech acts |
| parent#66 | parent | L | Med | Med | No — CLAUDE.md RAG-able content extraction across all repos |

---

### ── CODE (Java) — S scale ─────────────────────────────────────────────

**S / Low / High — do first, correctness bugs:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#342 | engine | fix: CaseLedgerEntry.traceId always null — @ObservesAsync severs OTel propagation | No |
| engine#331 | engine | fix: CapabilityTarget PlanItem stays RUNNING when Quartz retries exhausted | No |
| engine#248 | engine | fix(blackboard): JPA SubCaseGroupRepository atomicity — conditional UPDATE | No |

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
| qhorus#199 | qhorus | design: TrustGateService injectable but never called — COMMAND accepts any obligor | No |
| qhorus#197 | qhorus | design: EVENT telemetry in DB only — invisible in distributed traces | No |
| eidos#8 | eidos | design: EpistemicallyWeak should be soft preference signal, not hard filter | No |

**S / Med / Med — test reliability and code cleanup:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#303 | engine | test: SpiWiringIntegrationTest timing-sensitive, fails without Docker | No |
| engine#298 | engine | test: HumanTaskScheduleHandlerTest pre-existing isolation failures | No |
| qhorus#150 | qhorus | fix: A2AResource JSON error body, deriveState() ordering, ensureRegistered race | No |

**S — blocked:**

| Issue | Repo | What | Blocked by |
|-------|------|------|-----------|
| work#225 | work | feat: wire ExpiryLifecycleService to CaseSignalSink | ~~engine#349~~ ✅ **NOW UNBLOCKED** |
| work#221 | work | design: add SIGNAL_RECEIVED to WorkEventType | ~~engine#349~~ ✅ **NOW UNBLOCKED** |
| qhorus#200 | qhorus | feat: WatchdogAlertEvent + ConnectorAlertBridge | ~~engine#349~~ ✅ **NOW UNBLOCKED** |
| clinical#28 | clinical | fix: AeEscalationListener uses engine.internal.event | Engine must promote CaseLifecycleEvent to public SPI — no issue filed yet |

---

### ── CODE (Java) — M scale ─────────────────────────────────────────────

**M / Low / High — high impact, clear path:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#338 | engine | fix: WorkItemLifecycleAdapter collapses REJECTED/EXPIRED/ESCALATED → FAULTED | No |
| engine#339 | engine | fix: SpawnGroup M-of-N outcomes never reach engine (missing observer) | No |
| clinical#26 | clinical | feat: engine_case_id on ProtocolDeviation and AdverseEvent | No |
| clinical#27 | clinical | feat: AdverseEvent.escalationStatus field | No |
| work#229 | work | chore: rename db/migration/ → db/work/migration/ across all submodules | No — but blocks consumer updates in aml, clinical, devtown |

**M / Med / High — important, some design:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#337 | engine | design: WorkOrchestrator resolve WorkerSelectionStrategy via CDI | No — unblocks #336 |
| engine#336 | engine | design: SelectionContext carry trust scores | Blocked on #337 |
| engine#256 | engine | Expose CaseContext factory on SPI — decouple work-adapter | No |
| engine#274 | engine | feat(blackboard): hydrate BlackboardRegistry from PlanItemStore on restart | No |
| engine#212 | engine | feat: casehub-work multi-instance WorkItem spawning | No |
| qhorus#166 | qhorus | feat: dispatch MessageObserver after transaction commit via JTA | No |
| claudony#116 | claudony | feat: enable reactive Qhorus stack for PostgreSQL | No |

**M / Low / Med — mechanical but multi-file:**

| Issue | Repo | What | Blocked? |
|-------|------|------|----------|
| engine#325 | engine | HumanTaskTarget: add claimDeadlineHours field | No |
| engine#326 | engine | CasePlanModel YAML: failure goal support | No |
| engine#327 | engine | HumanTaskTarget: runtime-evaluated expiresIn | No |
| engine#302 | engine | feat: align startCase to accept any serializable input | No |
| work#191 | work | feat: extract persistence-memory module from testing/ | No |
| work#190 | work | migrate: VocabularyScope enum alignment with casehub-platform-api | No |
| work#189 | work | migrate: LabelDefinition.path → casehub-platform-api Path | No |
| work#175 | work | feat: JSON merge semantics for WorkItemTemplate.defaultPayload | No |
| work#184 | work | feat: excludedGroups — group-level conflict-of-interest exclusion | No |
| work#177 | work | feat: conditional outcomes (OHT condition expression) | No |
| qhorus#183 | qhorus | feat: add recovered flag to ChannelInitialisedEvent | No |
| qhorus#169 | qhorus | feat: extract persistence-memory module from testing/ | No |
| qhorus#158 | qhorus | feat: DefaultInboundNormaliser infer MessageType from correlationId | No |
| claudony#94 | claudony | feat: set causedByEntryId at provisioning time | No |
| claudony#125 | claudony | feat: SSE /api/mesh/events has no reconnect cursor | No |
| clinical#22 | clinical | feat: sponsor notification config endpoint | No |
| clinical#23 | clinical | feat: PI actor ID to display name resolution | No |
| connectors#1 | connectors | Connectors using MCP must follow normative patterns | No |
| connectors#2 | connectors | feat: Slack ChannelBackend — first external channel transport | No — unblocks clinical#11 |
| parent#66 | parent | CLAUDE.md review: extract RAG-able content under 40KB | No |

---

### ── CODE (Java) — L / XL scale ────────────────────────────────────────

| Issue | Repo | Scale | Complexity | Impact | Blocked? |
|-------|------|-------|------------|--------|----------|
| engine#349 | engine | ✅ DONE | — | — | — |
| engine#340 | engine | L | High | High | No |
| qhorus#162 | qhorus | L | High | High | No |
| qhorus#132 | qhorus | L | Med | High | No |
| claudony#102 | claudony | L | Med | High | No |
| claudony#98 | claudony | L | Med | High | No |
| connectors#4 | connectors | L | High | High | ~~engine#349~~ ✅ **NOW UNBLOCKED** |
| clinical#21 | clinical | L | Med | High | No |
| clinical#11 | clinical | L | Med | High | Blocked on connectors#2 |
| parent#56 | parent | L | High | High | No |
| parent#5 | parent | L | High | High | No |
| parent#3 | parent | L | High | Med | No |
| parent#13 | parent | L | Med | Med | No |

---

## Blocking Chain Summary

```
engine#349 ✅ DONE → unblocks:
  work#225 (ExpiryLifecycleService → CaseSignalSink)
  work#221 (SIGNAL_RECEIVED in WorkEventType)
  qhorus#200 (WatchdogAlertEvent + ConnectorAlertBridge)
  connectors#4 (inbound connector)
  parent#64 (partially — also needs work#225 + qhorus#200)

engine#337 (CDI strategy resolution) → unblocks engine#336 (trust scores)
connectors#2 (Slack ChannelBackend) → unblocks clinical#11 (AE notifications)
work#229 (db/migration rename) → triggers: aml, clinical, devtown config updates (classpath)

clinical#28 needs: engine to promote CaseLifecycleEvent to public SPI (file engine issue)
parent#64 fully unblocked when: work#225 ✅ + qhorus#200 ✅
```

---

## Key References

- Engine PR merged: casehubio/engine#368 (engine#367 + engine#350)
- Protocols written: PP-20260525-607b33 (Flyway repo-scoped paths), PP-20260526-6d39e5 (opaque cross-module identifiers)
- work#229 filed: db/work/migration rename — coordinate with aml, clinical, devtown
- Workspace isolation fixed: life/ and openclaw/ now own repos (wsp-casehub-life, wsp-casehub-openclaw)
- Plans untracked: `plans/2026-05-19-*.md` — pre-existing, not blocking
