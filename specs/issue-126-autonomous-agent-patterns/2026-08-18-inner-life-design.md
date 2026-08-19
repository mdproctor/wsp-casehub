# InnerLife Pattern — Design Spec

**Issue:** casehubio/blocks#119
**Date:** 2026-08-18
**Branch:** issue-126-autonomous-agent-patterns

## Summary

InnerLife is a background thought loop in `io.casehub.blocks.agentic.personality` that enables agents to initiate conversations proactively rather than only responding when spoken to. It composes existing platform capabilities (ReflectionOrchestrator, ObservationAccumulator, AffordanceRenderer, AgentProvider) into a three-stage evaluation pipeline gated by composable civility constraints.

The pattern produces initiation decisions with content — consuming apps control dispatch. It does NOT own channel communication, scheduling, or message formatting.

## Architecture

### Existing Platform Capabilities (composed, not built)

1. **ReflectionOrchestrator** (neocortex) — `reflect(agentId, tenantId, since, maxSourceMemories) → List<String>` — generates reflective thoughts from accumulated memories
2. **ObservationAccumulator** (blocks/summarisation) — thread-safe event buffer with tiered rendering for LLM prompts
3. **AffordanceRenderer** (blocks/summarisation/affordance) — renders available actions per entity for LLM context
4. **AgentProvider** (platform-agent-api) — `invoke()` with structured output for LLM evaluation

### What InnerLife Adds

1. **CivilityConstraint SPI** — composable pre-dispatch gating for social norms (rate limiting, gap enforcement, cooldown)
2. **Content quality gate** — novelty scoring + observation count + quiet-period bypass (System 1/2 fast path)
3. **InnerLifeOrchestrator** — periodic tick cycle: civility → content quality → reflect → score motivation → output
4. **Token-level novelty scorer** — zero-cost content novelty detection

### Evaluation Pipeline

```
tick() → CivilityConstraint.permitInitiation()    [D4 — zero cost]
              | permitted
         ContentQualityGate: novelty + obs count   [D3 — zero cost]
              | novel OR quiet-period bypass
         ReflectionOrchestrator.reflect()           [LLM call]
              | reflections
         AgentProvider.invoke() → motivation 0-1    [D2 — LLM call]
              | above threshold
         Initiated(content, channelHint, score)     [D1 — output]
```

Any stage that fails returns `Silent` immediately — no further cost.

### Relationship to Existing SPIs (D5, D6)

InnerLife is a blocks pattern (not a qhorus extension). It produces content dispatched VIA qhorus — it is a producer, not a qhorus component. CivilityConstraint is separate from ActivationRule — different architectural domains (social initiation vs orchestration activation). ActivationContext fields (activationCount, consecutiveIdleActivations, lastAggregationResult) reflect orchestration lifecycle, not social behaviour.

## CivilityConstraint SPI (D4)

### CivilityConstraint

```java
@FunctionalInterface
public interface CivilityConstraint {
    CivilityCheck permitInitiation(InitiationContext context);
}
```

### CivilityCheck

```java
public sealed interface CivilityCheck {
    record Permitted() implements CivilityCheck {}
    record Denied(String reason) implements CivilityCheck {}
}
```

### InitiationContext

```java
public record InitiationContext(
    Instant lastInitiationTimestamp,
    int initiationsInWindow,
    int consecutiveInitiationsWithoutResponse,
    AgentRef agent) {}
```

### Default Implementations

| Implementation | Default | Logic |
|---|---|---|
| `MinimumGapConstraint` | 5 min | Denied if `now - lastInitiation < gap` |
| `MaxPerWindowConstraint` | 3/hour | Denied if `initiationsInWindow >= max` |
| `ConsecutiveInitiationCooldownConstraint` | 2 consecutive | Denied if `consecutiveWithoutResponse >= max` (cooldown until someone else posts) |

The orchestrator runs all constraints — first `Denied` wins. All checked before any LLM call.

## Content Quality Gate (D3)

### ContentQualityGate

```java
public record ContentQualityGate(double noveltyThreshold,
                                  int minObservations,
                                  Duration quietPeriodBypass) {
    public static final double DEFAULT_NOVELTY_THRESHOLD = 0.3;
    public static final int DEFAULT_MIN_OBSERVATIONS = 3;
    public static final Duration DEFAULT_QUIET_PERIOD = Duration.ofMinutes(30);

    public static ContentQualityGate defaults() {
        return new ContentQualityGate(DEFAULT_NOVELTY_THRESHOLD,
                DEFAULT_MIN_OBSERVATIONS, DEFAULT_QUIET_PERIOD);
    }
}
```

### Evaluation Logic

1. If `observationCount < minObservations` AND `timeSinceLastLlmEval < quietPeriodBypass` → skip (Silent)
2. Compute token-level Jaccard distance between current observation text and previous observation text
3. If `novelty < noveltyThreshold` AND `timeSinceLastLlmEval < quietPeriodBypass` → skip (Silent)
4. Otherwise → proceed to reflection + LLM scoring

The quiet-period bypass enables spontaneous initiation: after extended silence, the LLM decides whether elapsed time alone justifies speaking. Without it, tick() would be functionally reactive.

### TokenJaccardDistance

Package-private utility. Token-level Jaccard distance: `1 - |A ∩ B| / |A ∪ B|` where A and B are whitespace-tokenized word sets from the observation texts. Returns 1.0 for completely disjoint texts, 0.0 for identical texts.

Note: qhorus's `JaccardSimilarity` is package-private and inaccessible from blocks. This is an independent implementation. Designed to be replaceable — embedding-based novelty via `CbrSimilarityScorer` (neocortex-memory-api) is the natural upgrade when semantic precision justifies the added dependency and latency.

## Orchestrator

### InnerLifeOrchestrator

`@ApplicationScoped` CDI bean following the established `tick()` pattern.

```java
@ApplicationScoped
public class InnerLifeOrchestrator {

    InnerLifeOrchestrator(
        ReflectionOrchestrator reflectionOrchestrator,
        AgentProvider agentProvider,
        Instance<CivilityConstraint> civilityConstraints,
        InnerLifeConfig config);

    void observe(LevelEvent<?> event, String agentId);

    void observeResponse(String agentId);

    InnerLifeTick tick(AgentDescriptor descriptor, String channelContext);
}
```

### observe(event, agentId)

Accumulates observations into per-agent `ObservationAccumulator`. Called by consuming app whenever channel activity occurs. Tracks observation count since last initiation.

### observeResponse(agentId)

Called when a non-self message is observed after an initiation. Resets `consecutiveInitiationsWithoutResponse` to 0. This is how the cooldown constraint knows someone responded.

### tick(descriptor, channelContext)

The periodic thought cycle:

1. **Civility gate (D4):** Build `InitiationContext` from per-agent state. Run all `CivilityConstraint` instances. First `Denied` → return `Silent(reason)`.
2. **Content quality gate (D3):** Check novelty + observation count + quiet-period bypass. Fail → return `Silent`.
3. **Reflect:** Call `reflectionOrchestrator.reflect(agentId, tenantId, since, maxSourceMemories)` → `List<String>` reflections.
4. **Score motivation (D2):** Call `agentProvider.invoke()` with structured output. Prompt includes: accumulated observations (drained from ObservationAccumulator), reflections, affordance context (from `channelContext` parameter), and agent personality (from descriptor). Returns `MotivationAssessment(double score, String content, String channelHint)`.
5. **Threshold check:** If `score >= motivationThreshold` → return `Initiated(content, channelHint, score)`. Otherwise → return `Silent`.
6. **Update state:** Record `lastLlmEvaluationTimestamp`, store current observation text as `previousObservationText`. If Initiated: record `lastInitiationTimestamp`, increment `initiationsInWindow` and `consecutiveInitiationsWithoutResponse`, reset observation count.

### InnerLifeTick (D1)

```java
public sealed interface InnerLifeTick {
    record Silent(@Nullable String reason) implements InnerLifeTick {}
    record Initiated(String content, @Nullable String channelHint,
                     double motivationScore) implements InnerLifeTick {}
}
```

`channelHint` is the LLM's suggestion based on affordance context provided by the consumer via the `channelContext` parameter. The LLM reasons about channel names in observation text and suggests a target. No independent channel topology dependency.

### Per-Agent State

In-memory, keyed by `tenancyId:agentId` in `ConcurrentHashMap`:

| Field | Purpose |
|---|---|
| `ObservationAccumulator` | Buffered observations |
| `lastInitiationTimestamp` | For civility gap check |
| `lastLlmEvaluationTimestamp` | For quiet-period bypass |
| `initiationsInWindow` | For rate limiting (window resets on configurable interval) |
| `consecutiveInitiationsWithoutResponse` | For cooldown (reset via `observeResponse()`) |
| `previousObservationText` | For novelty scoring (Jaccard distance vs current) |
| `observationCountSinceLastInitiation` | For minimum observation check |

**Restart resilience:** All state is in-memory (`@ApplicationScoped` lifecycle). On restart, all agents start with zero state — no initiations pending, no cooldowns active. This is acceptable because the first tick after restart evaluates fresh, and the conservative defaults (5 min gap, 3/hour max) prevent over-posting even without history. Same reasoning as PersonalityEvolutionOrchestrator.

### Thread Safety

Per-agent `ReentrantLock` for `tick()` (same pattern as PersonalityEvolutionOrchestrator). `observe()` delegates to synchronized `ObservationAccumulator.collect()`. `observeResponse()` uses atomic operations on the per-agent state.

### Configuration (InnerLifeConfig)

| Key | Default | Meaning |
|---|---|---|
| `motivationThreshold` | 0.6 | LLM motivation score required to initiate |
| `noveltyThreshold` | 0.3 | Minimum Jaccard distance for content novelty |
| `minObservations` | 3 | Minimum observations before LLM evaluation |
| `quietPeriodBypass` | 30 min | Time before quiet-period triggers LLM evaluation regardless |
| `maxReflectionSources` | 10 | Max source memories for ReflectionOrchestrator |
| `windowDuration` | 1 hour | Time window for MaxPerWindowConstraint rate limiting |

## Package Structure

**`io.casehub.blocks.agentic.personality`** (additions to existing package)

| Class | What it does |
|---|---|
| `CivilityConstraint` | `@FunctionalInterface` SPI: `permitInitiation(InitiationContext) → CivilityCheck` |
| `CivilityCheck` | Sealed: `Permitted`, `Denied(reason)` |
| `InitiationContext` | Record: lastInitiationTimestamp, initiationsInWindow, consecutiveInitiationsWithoutResponse, agent |
| `MinimumGapConstraint` | Default civility: denied if gap too short |
| `MaxPerWindowConstraint` | Default civility: denied if rate exceeded |
| `ConsecutiveInitiationCooldownConstraint` | Default civility: denied after N unanswered initiations |
| `ContentQualityGate` | Record: novelty threshold, min observations, quiet-period bypass |
| `InnerLifeTick` | Sealed: `Silent(reason)`, `Initiated(content, channelHint, motivationScore)` |
| `InnerLifeOrchestrator` | CDI bean: `observe()`, `observeResponse()`, `tick()` |
| `InnerLifeConfig` | Record with defaults and preference-based resolution |
| `TokenJaccardDistance` | Package-private: token-level novelty scoring |

## Testing Strategy

All tests are plain JUnit 5 + Mockito (no Quarkus runtime).

| Test class | Coverage |
|---|---|
| `CivilityConstraintTest` | All three default constraints: gap, rate, cooldown — boundary values, edge cases |
| `ContentQualityGateTest` | Novelty scoring, observation count, quiet-period bypass, combined evaluation |
| `TokenJaccardDistanceTest` | Token-level similarity: identical text, disjoint text, partial overlap, empty input |
| `InnerLifeOrchestratorTest` | Full pipeline: civility denied → Silent, low novelty → Silent, low motivation → Silent, high motivation → Initiated. Mock ReflectionOrchestrator and AgentProvider. |
| `InnerLifeTickTest` | Sealed type exhaustiveness, field access |
| `ObserveAndStateTest` | Observation accumulation, state tracking (lastInitiation, consecutiveWithoutResponse reset on non-self message) |

## References

- `ReflectionOrchestrator` (neocortex) — reflection generation SPI
- `ObservationAccumulator` (blocks/summarisation/observation) — event buffering with tiered rendering
- `AffordanceRenderer` (blocks/summarisation/observation/affordance) — action vocabulary rendering
- `ActivationRule` / `ActivationContext` (blocks/agentic/activation) — separate domain (D6)
- `PersonalityEvolutionOrchestrator` (blocks/agentic/personality) — established tick() pattern
- `AgentProvider.invoke()` (platform-agent-api) — LLM invocation with structured output
- `Watchdog` / `WatchdogConditionType` (qhorus-api) — channel-level alerting (distinct from agent-level civility, D5)
- Research doc §2.1 — InnerLife pattern description, capability composition
- Research doc §2.8 — System 1/2 fast path fold-in, novelty engine fold-in
- Liu, Fang, Shi, Wu, Igarashi, Chen — "Proactive Conversational Agents with Inner Thoughts" (CHI 2025, arXiv:2501.00383) — 82% user preference for agents with inner thoughts
- Deng et al. — "Human-Centered Proactive Agents: Intelligence, Adaptivity, Civility" (AAAI 2025) — civility constraints taxonomy
- DPT-Agent (2025) — Dual Process Theory with Theory of Mind
