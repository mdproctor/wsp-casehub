# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-18g:** Housekeeping session — closed 6 issues directly on main (3 already-done, 3 with code changes). Code reviewed, blog written, 2 follow-up issues filed (#1148, #1149). ARC42STORIES deployment contract strengthened after review. All commits on main, not a feature branch.

**Queue position:** 70/78 (8 open items + 1 deferred). Next: #965.

## What Was Done — Session 18g

### Commits on neocortex main

| SHA | Issue | Summary |
|-----|-------|---------|
| `e8e1504a5` | #982 | CDI `Instance<SessionPipeline>` in ConsolidationScheduler — 6 phases were receiving null |
| `fcb7ed89d` | #1104 | Single-instance-per-tenant deployment contract in ARC42STORIES §7 + §12 |
| `3cb95e55a` | #574 | Phase 2b provenance fields: `related_entities`, `source_entity_ref`, `source_provenance` |
| `988a9035d` | #962 | Instrument 4 remaining CDE methods with derivation journal (14/14 coverage) |
| `1bbf56bd1` | #1104 | Add 6 cognitive orchestrators to deployment contract (code review finding) |

### Issues Closed (no code change — already done)

- **#1008** — artifact_sweep.py verification module import (fix already applied)
- **#541** — GOAL_SYSTEM extraction prompt (already updated)

### Code Review Findings

1. **Fixed:** ARC42STORIES deployment contract missing 6 orchestrators (MoodOrchestrator, DriveOrchestrator, NarrativeOrchestrator, GroupNarrativeOrchestrator, CognitiveGoalOrchestrator, SocialNormDetector) → committed as `1bbf56bd1`
2. **Filed #1148:** coverage.py `source_provenance` mapping doesn't consume Phase 2b's `source_provenance` field
3. **Filed #1149:** Literature citations (Eysenck, McAdams, Paulhus & Williams, etc.) removed during CDE instrumentation refactoring

### Blog

- `blog/2026-11-18-mdp01-the-pipeline-that-couldnt-explain-itself.md` — four traceability fixes: CDI wiring, deployment contracts, data lineage, derivation journal

### Issues Filed

- **casehubio/neocortex#1148** — coverage: source_provenance field not consumed for Phase 2b (XS/Low)
- **casehubio/neocortex#1149** — docs: restore literature citations in CDE instrumented methods (XS/Low)

### Issues Closed

- **casehubio/neocortex#1008** — artifact_sweep.py crashes on image co-promotion
- **casehubio/neocortex#982** — consolidation: setSessionPipeline propagation is dead
- **casehubio/neocortex#1104** — Deployment contract: single-instance per tenant
- **casehubio/neocortex#574** — Phase 2b memories lack provenance fields
- **casehubio/neocortex#541** — Update GOAL_SYSTEM extraction prompt
- **casehubio/neocortex#962** — Instrument remaining 4 CDE methods

## Queue — remaining items

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #965 | GraphMutated CDI listener — agentId | S | Med | ← next. Design choice: ThreadLocal vs event enrichment |
| #966 | Instrument ConsolidationScheduler phases | S | Low | Pairs with #962 (CDE done, scheduler phases not) |
| #981 | Wire journal bridges — production | S | Med | |
| #967 | Evaluate CognitiveJournal lower layer | S | Med | |
| #1067 | Extract PredictionCalibrationAnalyzer | S | Low | |
| #1066 | Expand PersonalityDifferentiationTest | S | Low | |
| #1148 | coverage source_provenance Phase 2b | XS | Low | Filed this session |
| #1149 | Restore CDE literature citations | XS | Low | Filed this session |

### Deferred

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1089 | Unbounded cache growth situational personality | XS | Low |

## Prior Session — 18f

Completed #1133 (tomFacetScore per-facet defaults) and #1134 (CognitionObservationBuilder personality-modulated ToM). Filed #1137.

## Neocortex Build Note

Slot repos may need syncing with canonical before building. Run:
```bash
git -C neocortex fetch canonical main && git -C neocortex rebase canonical/main
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/pom.xml install -Dmaven.test.skip=true -s neocortex/.mvn/slot-settings.xml -T1C
```

## IntelliJ Slot Note

IntelliJ MCP may become unresponsive during heavy rebase/indexing. If `ide_index_status` shows unresponsive for >60s, restart IDE. Use python scripts in scratchpad as fallback for file writes when IDE is down.
