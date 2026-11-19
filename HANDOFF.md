# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-19b:** Completed 12 issues across two sub-sessions. All work published to origin. Queue has 3 open items + 1 deferred. Protocol captured for queue routing.

**Queue position:** 82/86 (3 open + 1 deferred). Next: #1160.

## What Was Done — Session 19a+19b

### Issues Closed (code changes)

| SHA | Issue | Summary |
|-----|-------|---------|
| `23d84dbcc` | #966 | Per-phase `Consolidated` journal entries |
| `938a0513f` | #966 | Defensive try-catch guard |
| `cf268f3ee` | #1150 | agentId across `@ObservesAsync` |
| `478d6a78e` | #1151 | ConversationBridge agentId from principalId |
| `0a149c15a` | #981 | Wire journal bridges in production |
| `eb0447d7a` | #1067 | Extract PredictionCalibrationAnalyzer to cognition-api |
| `1f1c592bf` | #1148 | coverage.py source_provenance mapping |
| `e88f7fda1` | #1149 | Restore CDE literature citations |
| `0f3556809` | #1155 | Tick counter for EventRef |
| `12f689e4a` | #1156 | inputNodeCount/outputNodeCount on ConsolidationPhase |
| `13e29be55` | #1159 | Per-agent curiosity deprivation bias |

### Issues Closed (evaluation/investigation)

| Issue | Decision |
|-------|----------|
| #967 | Keep bridge pattern |
| #1066 | Harness blockers identified |
| #1161 | Closed — serial tunnel rejected; ScopedValue filed as #1162 |

### Protocol Captured

**PP-20261119-8971fd** — Always append to primary workspace queue (`wsp-casehub/.queue`). Per-repo workspace queues are invisible to work.py.

### Publishing

- neocortex: 14 commits published to origin
- examples: 1 commit published to origin (protocol)

## Queue — remaining items

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1160 | CognitionTestHarness CAPS/behavioral attractors | M | Med |
| #1162 | MutationContext → ScopedValue migration | S | Low |
| #1163 | EffectiveCognitiveDefaultsTest situational origin fix | XS | Low |

### Deferred

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1089 | Unbounded cache growth situational personality | XS | Low |

### Key Decision: ThreadLocal vs ScopedValue

MutationContext uses ThreadLocal for cross-cutting mutation provenance. This is the correct pattern (not serial tunnel through 30+ classes). ThreadLocal breaks at `@ObservesAsync` — the agentId on ExtractionRequested handles that boundary. #1162 proposes migrating to `ScopedValue` (JEP 506, final in JDK 25) when JDK 25+ runtime is confirmed. ScopedValue is immutable, auto-scoped, no cleanup burden.

## Neocortex Build Note

```bash
git -C neocortex fetch canonical main && git -C neocortex rebase canonical/main
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/pom.xml install -Dmaven.test.skip=true -s neocortex/.mvn/slot-settings.xml -T1C
```

## Known Pre-existing Test Failure

`EffectiveCognitiveDefaultsTest.situationalOriginSetWhenFacetShifted` — filed as #1163.
