# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-18d:** Completed #1125 (cache eviction scheduler). Filed #1131 (thread-safety hardening) and examples#211 (wacky-manor wiring). Blog written. Work landed and published to remote.

**Slot 196 is marked ready for archive** — queue has remaining items but all branches landed.

**Next issue:** #1127 — Suspiciousness-modulated evidence weighting for TheoryOfMindEngine (S/Med). Needs a fresh branch.

## What Was Done — Session 18d

### Landed commit `be9996c7b` (neocortex, published)

**feat(#1125): wire periodic evictIdle() calls for cognitive caches**

- `CacheEvictionScheduler` — dual-threshold (tick count AND wall-clock time). Evicts only when both >= 60 ticks AND >= 5 minutes since last eviction. Registered evictors receive configurable `maxIdle` duration (default 10 min).
- Wired into `CognitionCore.tick()` at end of pipeline, after snapshot store.
- `EffectiveCognitiveDefaults` auto-registers via `configureCacheEviction()`.
- `SituationalPersonalityParticipant` registered externally (not wired in any app yet — see examples#211).
- Builder support: `CognitivePipelineBuilder.cacheEviction(scheduler)`.
- Review fixes: try-catch per evictor (exception resilience), removed double-registration path from `setEffectiveDefaults`.

### Tests (11 total)

- 7 unit tests: tick-only threshold, time-only threshold, both thresholds, reset after eviction, no-op with no evictors, multi-evictor, default constructor, exception resilience
- 3 integration tests: scheduler invoked during tick, effectiveDefaults auto-registration, null-safe when unconfigured

### Blog

- `blog/2026-11-18-mdp03-the-methods-nobody-called.md` — dual-threshold design, configure-or-not pattern, review catches

### Issues Filed

- **casehubio/neocortex#1131** — `CacheEvictionScheduler` use `CopyOnWriteArrayList` for thread safety (XS/Low)
- **casehubio/examples#211** — Wire `SituationalPersonalityParticipant.evictIdle` in wacky-manor (XS/Low)

## Prior Sessions (retained for context)

### Session 18c — branch `issue-1097-personality-modulated-tom-cpts` (landed)

1. **feat (#1097):** `TomPersonalityModulation` — anxiety shifts threat prior (Bar-Haim 2007), callousness weakens distress-avoidant coupling (Blair 2005/2013).
2. **feat (#1089):** `evictIdle(Duration)` on situational caches.

### Session 18b — branch `issue-1113-remove-keyword-sec-checks` (landed)

1. **feat (#1113):** Deleted all keyword-based SEC checks.
2. **feat (#1112):** `ValenceClassificationEvalTest` — 14 BDI scenarios.

### Session 18a — branch `issue-1098-bdi-semantic-classifier` (landed)

1. **feat (#1098):** `IntentValence` enum, LLM extraction prompt valence, removed keyword classifiers.

## Queue — remaining items

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1127 | Suspiciousness-modulated ToM evidence weighting | S | Med | Active — next to start |
| #1131 | CacheEvictionScheduler CopyOnWriteArrayList | XS | Low | Thread-safety hardening |

### Deferred

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1089 | Unbounded cache growth situational personality | XS | Low | Partially addressed by #1125 scheduler |

## Neocortex Build Note

Slot repos may need syncing with canonical before building. Run:
```bash
git -C neocortex fetch canonical main && git -C neocortex rebase canonical/main
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/pom.xml install -Dmaven.test.skip=true -s neocortex/.mvn/slot-settings.xml -T1C
```

## IntelliJ Slot Note

IntelliJ MCP may become unresponsive during heavy rebase/indexing. If `ide_index_status` shows unresponsive for >60s, restart IDE. Use python scripts in scratchpad as fallback for file writes when IDE is down.
