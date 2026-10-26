# Design Journal — main

## 2026-10-26 — Epic #149 pre-run cognitive readiness: 3 issues landed

### #131 — Externalize clearAtTokens

SessionPipeline's context budget was hardcoded to 80,000 tokens across three call sites. Extracted to `manor.session.clear-at-tokens` with format parsing: plain number, k-suffix (`80k`), or percentage of model context window (`40%`). The percentage format requires `manor.session.model-context-tokens` (default 200,000). Parsing lives as a static method on `ManorConfig.SessionConfig` for testability.

### #110 — Code review findings

Four warnings resolved: `CognitiveSnapshotRecorder` implements `AutoCloseable` with `finally`-guarded close; `ManorResource.getEvents()` replaced manual `replace()` escaping with Jackson `ObjectMapper`; `PlaybookOrchestrator` wraps `sessionPipeline` in `try-finally` for both AUTONOMOUS and SCRIPTED modes; `ManorConsolidationBeans` uses `@ConfigProperty` for the relationship stage interval instead of hardcoded `5 * 60_000L`.

### #142 — Per-character NeedSatisfactionConfig overrides

New `NeedSatisfactionConfigProvider` SPI in neocortex. `DriveAdaptationPhase` resolves per-agent config after `agentId` is known in `processSubgraph`. Manor implementation reads `SocialConfig.Disposition.socialOrient()` — competitive characters get 1.5x social decay rate and 0.6x social resting level. Dispositions populated from social configs at scenario start before first consolidation pass.

### Remaining

- `casehubio/neocortex#684` — Import goal_coherence as goal→goal edges
- `casehubio/examples#135` — Full scenario run with all cognitive components enabled
