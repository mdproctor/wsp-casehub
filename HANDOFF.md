# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-19a:** Completed 9 issues across the observability initiative. Queue fully drained (only deferred #1089 remains). Code reviewed, blog revised with 6 traceability layers, 4 follow-up issues filed (#1155, #1156, #1159, #1160, #1161). Research docs updated.

**Queue status:** drained. 1 deferred (#1089). 4 follow-up issues in neocortex workspace queue. Main queue empty.

## What Was Done — Session 19a (continuing from 18h)

### Issues Closed (code changes)

| SHA | Issue | Summary |
|-----|-------|---------|
| `23d84dbcc` | #966 | Per-phase `Consolidated` journal entries in ConsolidationScheduler |
| `938a0513f` | #966 | Defensive try-catch — journal failures cannot corrupt PhaseResult |
| `cf268f3ee` | #1150 | Add agentId to ExtractionRequested, propagate across `@ObservesAsync` |
| `478d6a78e` | #1151 | ConversationBridge.process() sets agentId from principalId |
| `0a149c15a` | #981 | Wire DerivationJournalBridge + MemoryRecallJournalBridge in production CDI |
| `eb0447d7a` | #1067 | Extract PredictionCalibrationAnalyzer/CalibrationReport/SourceOutcomeSummary to cognition-api |
| `1f1c592bf` | #1148 | coverage.py source_provenance maps to itself, not source_passage |
| `e88f7fda1` | #1149 | Restore literature citations in CDE instrumented methods |

### Issues Closed (evaluation/investigation, no code)

| Issue | Decision |
|-------|----------|
| #967 | Keep bridge pattern — moving CognitiveJournal to lower layer would require splitting CognitiveEntry types |
| #1066 | Harness blockers identified — per-agent curiosity wiring, CAPS exposure, behavioral attractor setup all missing |

### Follow-up Issues Filed

| Issue | Title | Scale | Complexity |
|-------|-------|-------|-----------|
| #1155 | ConsolidationScheduler tick counter for EventRef | XS | Low |
| #1156 | ConsolidationPhase input/output node counts | S | Low |
| #1159 | CognitionTestHarness per-agent curiosity wiring | S | Med |
| #1160 | CognitionTestHarness CAPS/behavioral attractor exposure | M | Med |
| #1161 | Replace MutationContext ThreadLocal with explicit parameter passing | M | Med |

### Code Review Findings

1. **Fixed:** journal.record() inside phase try-catch → duplicate PhaseResult. Wrapped defensively.
2. All CDI wiring uses Instance.isResolvable() consistently.
3. ExtractionRequested has 3 backward-compatible constructors (5/7/8 args).
4. MutationContext ThreadLocal identified as a broader problem → filed #1161.

### Blog

- `blog/2026-11-18-mdp01-the-pipeline-that-couldnt-explain-itself.md` — revised 3x. Final version has 7 sections, 6 traceability layers (SVG), covers scheduler journal, defensive pattern, bridge wiring, and ThreadLocal/@ObservesAsync async boundary gotcha.

## Remaining Queue Items (neocortex workspace only)

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1155 | ConsolidationScheduler tick counter EventRef | XS | Low |
| #1156 | ConsolidationPhase input/output node counts | S | Low |
| #1159 | CognitionTestHarness per-agent curiosity | S | Med |
| #1160 | CognitionTestHarness CAPS/attractors | M | Med |
| #1161 | Replace MutationContext ThreadLocal | M | Med |

### Deferred (main queue)

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1089 | Unbounded cache growth situational personality | XS | Low |

## Prior Session — 18g/18h

Closed 6 observability issues (#982, #1104, #574, #962, #1008, #541). Filed #1148, #1149. CDI SessionPipeline wiring, deployment contract, Phase 2b provenance, CDE journal instrumentation.

## Neocortex Build Note

Slot repos may need syncing with canonical before building. Run:
```bash
git -C neocortex fetch canonical main && git -C neocortex rebase canonical/main
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/pom.xml install -Dmaven.test.skip=true -s neocortex/.mvn/slot-settings.xml -T1C
```

## Known Pre-existing Test Failure

`EffectiveCognitiveDefaultsTest.situationalOriginSetWhenFacetShifted` — expects `"situational"` but gets `"situational [usual 0.50]"`. Unrelated to this session's work. Affects only `cognitive-index` tests.
