# PersonalityEvolution Pattern — Design Spec

**Issue:** casehubio/blocks#118
**Date:** 2026-08-18
**Branch:** issue-126-autonomous-agent-patterns

## Summary

PersonalityEvolution is a signal-driven orchestrator in `io.casehub.blocks.agentic.personality` that composes existing eidos and neocortex capabilities into a bounded personality drift feedback loop. Interaction outcomes (behavioral compliance, relationship quality, goal achievement) nudge personality traits within configurable ranges over time.

The pattern does NOT duplicate eidos's JPAF evaluation logic. It fills the gap between "interaction outcomes happen" and "the JPAF machinery processes them" by providing signal translation, periodic orchestration, and safety rails.

## Architecture

### Existing Platform Capabilities (composed, not built)

1. **DispositionSignalStore** (eidos-api) — accumulates activation counts per cognitive function per agent, with decay support
2. **DefaultDispositionHealth.probe()** (eidos-runtime) — computes effective weights from `baseWeight + activationCount × Δw`, detects evolution thresholds (auxiliary surpasses dominant, shadow surpasses primary, structural reorganization)
3. **DefaultDispositionEvolution.evaluate()** (eidos-runtime) — applies 4 JPAF decision rules (dominant swap, dominant replacement, auxiliary replacement, structural reorganization) and normalizes weights

### What PersonalityEvolution Adds

1. **Signal translation SPI** — `TraitPressureSource<E>` maps domain events to disposition function activations using a two-layer strategy
2. **Orchestrator** — `PersonalityEvolutionOrchestrator` drives the periodic tick cycle: decay → probe → evaluate → persist
3. **Safety rails** — L2 displacement ceiling with halt flag, probe-time asymmetric dampening for negative signals

### Data Flow

```
Domain events → TraitPressureSource.translate() → DispositionSignalStore.recordActivation(valence)
                                                          |
tick() → DispositionSignalStore.decay() → DispositionHealth.probe()
              |                                    |
         L2 ceiling check              EvolutionPending?
              |                           | yes
         halt flag <--------  DispositionEvolution.evaluate()
                                    |
                              Evolved → persist new profile, clear signals
                              Dampened → decay signals, leave halt flag
```

## Signal Translation SPI

### TraitPressureSource

```java
@FunctionalInterface
public interface TraitPressureSource<E> {
    List<TraitActivation> translate(E event, AgentDescriptor descriptor);
}
```

Returns zero or more `TraitActivation` records. Zero means the event is irrelevant to this agent's personality. The SPI receives the full `AgentDescriptor` for access to `dispositionProfile`, `dispositionVocabulary`, and `agentId`/`tenancyId`.

### TraitActivation

```java
public record TraitActivation(String functionTerm, SignalValence valence) {}
```

The `functionTerm` must match a term in the agent's `dispositionProfile`. Activations with unrecognised terms are silently dropped (defensive — profile may have changed between event and recording).

### SignalValence

```java
public enum SignalValence { POSITIVE, NEGATIVE }
```

Lives in eidos-api (shared between blocks and eidos). POSITIVE signals reinforce a function. NEGATIVE signals indicate compensatory pressure. At probe time, negative activations are weighted by the dampening factor (D4).

### Two-Layer Mapping (D2)

**Layer 1 — Profile-term-aware (universal):** The implementation reads `descriptor.disposition().dispositionProfile()` to get available terms. It maps the domain event to one or more of those terms. This works for ANY vocabulary framework — Big Five agents activate "openness"/"conscientiousness", DISC agents activate "dominance"/"influence", Jungian agents activate "ti"/"ne". Every agent with a populated `dispositionProfile` participates in personality evolution.

**Layer 2 — Vocabulary-structural (enrichment):** For agents whose vocabulary supports structural navigation (e.g., `JungianFunctionTerm.opposite()` returns the shadow function), the implementation additionally infers indirect activations. Example: a negative event on Ti also activates shadow Fe. This enrichment requires vocabulary-specific structural knowledge not derivable from profile terms alone. Agents without structural vocabulary simply skip Layer 2 — they still get Layer 1 evolution.

### Default Implementations

| Implementation | Event type | Mapping logic |
|---|---|---|
| `BehavioralSignalPressureSource` | `BehavioralSignal` | SUCCESS/COMPLIANT → POSITIVE activation of dominant function. DECLINE/VIOLATED → NEGATIVE activation of compensating function (vocabulary-structural if available). |
| `RelationshipPressureSource` | `RelationshipEvent` | POSITIVE quality → POSITIVE activation of social-oriented function. NEGATIVE quality → NEGATIVE activation. Function selection from profile terms by semantic match. |
| `GoalOutcomePressureSource` | `GoalOutcomeCounts` | High success rate → POSITIVE activation of dominant. Low success rate → NEGATIVE activation of compensating function. |

Consumer repos provide domain-specific implementations for events not covered by the defaults (e.g., conversation deadlock → activate conflict-mode function).

## Orchestrator

### PersonalityEvolutionOrchestrator

`@ApplicationScoped` CDI bean following blocks' established `tick()` pattern (SummarisationRunner, KeyedSummarisationRunner, ObservationAccumulator).

```java
@ApplicationScoped
public class PersonalityEvolutionOrchestrator {

    PersonalityEvolutionOrchestrator(
        DispositionSignalStore signalStore,
        DispositionHealth health,
        DispositionEvolution evolution,
        Instance<TraitPressureSource<?>> pressureSources);

    <E> void record(E event, AgentDescriptor descriptor);

    EvolutionTick tick(AgentDescriptor descriptor, ProbeContext probeContext);
}
```

### record(event, descriptor)

1. Check halt flag for this agent — if set, skip (L2 ceiling exceeded, recording paused)
2. Find matching `TraitPressureSource<E>` by event type (first match from CDI Instance)
3. Call `translate(event, descriptor)` → `List<TraitActivation>`
4. For each activation, call `signalStore.recordActivation(agentId, tenancyId, functionTerm, valence)`

### tick(descriptor, probeContext)

The periodic cycle, called by consuming apps at their chosen cadence (D3, D6):

1. **Decay:** `signalStore.decay(agentId, tenancyId, decayFactor)`
2. **Probe:** `health.probe(descriptor, probeContext)` → `DispositionStatus`
3. **Process result** (D5 halt flag state machine):

| DispositionStatus | Condition | Action | Return |
|---|---|---|---|
| `Aligned` | — | Clear halt flag | `EvolutionTick.Stable` |
| `Drifted` | `magnitude < l2Ceiling` | Clear halt flag | `EvolutionTick.Drifting(magnitude)` |
| `Drifted` | `magnitude >= l2Ceiling` | Set halt flag | `EvolutionTick.Halted(magnitude)` |
| `EvolutionPending` | `Evolved` result | Persist new profile, `signalStore.clear()`, clear halt flag | `EvolutionTick.Evolved(prev, new)` |
| `EvolutionPending` | `Dampened` result | `signalStore.decay(factor)`, leave halt flag | `EvolutionTick.Dampened(factor)` |

Note: `probe()` returns `Drifted` (not `EvolutionPending`) when the dominant exceeds `overReinforcementThreshold`, so `EvolutionPending` fires only when displacement has crossed an evolution threshold but NOT the over-reinforcement ceiling.

### EvolutionTick

Sealed return type giving callers precise information about what happened:

```java
public sealed interface EvolutionTick {
    record Stable() implements EvolutionTick {}
    record Drifting(double magnitude) implements EvolutionTick {}
    record Halted(double magnitude) implements EvolutionTick {}
    record Evolved(String previousType, String newType) implements EvolutionTick {}
    record Dampened(double decayFactor) implements EvolutionTick {}
}
```

### Thread Safety

Halt flag is an `AtomicBoolean` per agent, keyed in a `ConcurrentHashMap<String, AtomicBoolean>`. `record()` and `tick()` are safe for concurrent callers. `record()` is best-effort check-then-act — a race between halt check and recording is acceptable since the next `tick()` corrects it.

### Configuration

Via platform `PreferenceProvider`, per-tenant (consistent with eidos's existing `DispositionPreferenceKeys`):

| Key | Default | Meaning |
|---|---|---|
| `decayFactor` | 0.8 | Retention fraction per tick (0.8 = 20% decay per tick) |
| `l2Ceiling` | 0.15 | Maximum L2 displacement from baseline before halt |
| `dampeningFactor` | 0.5 | Weight multiplier for NEGATIVE activations at probe time |

Cascaded attenuation with defaults: an agent receiving one negative activation per tick contributes `1 × 0.5 (dampening) × 0.06 (reinforcementDelta) = 0.03` to effective weight per tick, decaying by 20% each subsequent tick. This produces slow, bounded drift as intended.

## Eidos SPI Enhancement

`DispositionSignalStore` gains two default methods for valence-aware storage. All new methods have defaults, so existing implementations (JPA, InMemory, NoOp) continue unchanged.

### New Types (eidos-api)

```java
// SignalValence enum (shared)
public enum SignalValence { POSITIVE, NEGATIVE }

// ValenceCounts record
public record ValenceCounts(int positive, int negative) {
    public int effective(double dampeningFactor) {
        return positive + (int) Math.round(negative * dampeningFactor);
    }
}
```

### DispositionSignalStore Additions

```java
public interface DispositionSignalStore {
    // Existing — unchanged
    void recordActivation(String agentId, String tenancyId, String functionTerm);
    Map<String, Integer> activationCounts(String agentId, String tenancyId);
    void decay(String agentId, String tenancyId, double decayFactor);
    void clear(String agentId, String tenancyId);

    // New — valence-aware recording (default delegates to existing)
    default void recordActivation(String agentId, String tenancyId,
                                   String functionTerm, SignalValence valence) {
        recordActivation(agentId, tenancyId, functionTerm);
    }

    // New — valence-split counts for probe-time dampening
    default Map<String, ValenceCounts> valenceCounts(String agentId, String tenancyId) {
        var counts = activationCounts(agentId, tenancyId);
        var result = new java.util.LinkedHashMap<String, ValenceCounts>();
        counts.forEach((k, v) -> result.put(k, new ValenceCounts(v, 0)));
        return result;
    }
}
```

### DefaultDispositionHealth.probe() Evolution

`probe()` evolves to use `valenceCounts()` instead of `activationCounts()`:

```java
// Before:
final double raw = dv.weight() + counts.getOrDefault(dv.term(), 0) * delta;

// After:
final var vc = valenceCounts.getOrDefault(dv.term(), new ValenceCounts(0, 0));
final double raw = dv.weight() + vc.effective(dampeningFactor) * delta;
```

The `dampeningFactor` is resolved from preferences alongside the existing `reinforcementDelta` and `overReinforcementThreshold`.

## Complementary Safety Mechanisms

Two independent safety mechanisms protect against runaway drift:

| Mechanism | Owner | Scope | Trigger |
|---|---|---|---|
| `overReinforcementThreshold` | eidos (existing) | Per-function | Dominant effective weight exceeds ceiling (default 0.50) |
| `l2Ceiling` | blocks (new) | Global profile | L2 distance from baseline exceeds ceiling (default 0.15) |

These are distinct geometric properties. The per-function threshold catches one function becoming disproportionately strong. The L2 ceiling catches distributed drift across multiple dimensions that no single per-function ceiling would detect. Both fire independently.

## Package Structure

**`io.casehub.blocks.agentic.personality`** (new sub-package)

| Class | What it does |
|---|---|
| `TraitPressureSource<E>` | `@FunctionalInterface` SPI: domain event → trait activations |
| `TraitActivation` | Record: `(String functionTerm, SignalValence valence)` |
| `EvolutionTick` | Sealed: Stable, Drifting, Halted, Evolved, Dampened |
| `PersonalityEvolutionOrchestrator` | CDI bean: `record()`, `tick()` |
| `PersonalityEvolutionConfig` | Record with defaults and preference-based resolution |
| `BehavioralSignalPressureSource` | Default source for `BehavioralSignal` |
| `RelationshipPressureSource` | Default source for `RelationshipEvent` |
| `GoalOutcomePressureSource` | Default source for `GoalOutcomeCounts` |

**Eidos-api additions:**

| Class | What it does |
|---|---|
| `SignalValence` | Enum: POSITIVE, NEGATIVE |
| `ValenceCounts` | Record with `effective(dampeningFactor)` |
| `DispositionSignalStore` | Two new default methods |

## Testing Strategy

All tests are plain JUnit 5 + Mockito (no Quarkus runtime).

| Test class | Coverage |
|---|---|
| `PersonalityEvolutionOrchestratorTest` | Core tick cycle: Stable/Drifting/Halted/Evolved/Dampened outcomes with mock signalStore, health, evolution |
| `HaltFlagTest` | Recording stops when L2 ceiling exceeded, resumes after decay brings displacement within bounds |
| `BehavioralSignalPressureSourceTest` | Signal mapping for each BehavioralSignal value across Big Five, DISC, and Jungian profiles |
| `RelationshipPressureSourceTest` | Quality signal mapping across vocabulary types |
| `GoalOutcomePressureSourceTest` | Success/failure rate mapping with threshold boundaries |
| `ValenceCountsTest` | `effective()` computation with various dampening factors, edge cases (zero counts, zero dampening) |
| `EvolutionTickTest` | Sealed type exhaustiveness, field access |

## References

- `DefaultDispositionHealth.java` (eidos) — probe logic, L2 computation, evolution threshold detection
- `DefaultDispositionEvolution.java` (eidos) — JPAF decision rules implementation
- `DispositionSignalStore.java` (eidos) — activation signal accumulation SPI
- `VocabularyTerm.opposite()` (eidos-api) — vocabulary-generic structural navigation
- `SummarisationRunner.tick()` (blocks) — established tick() integration pattern
- `GE-20260811-e941cc` — AgentDisposition vs DispositionProfile type distinction
- `GE-20260728-a53632` — vocabulary-generic structural navigation technique
- `GE-20260729-172d18` — DefaultDispositionEvolution implements 4 JPAF decision rules
- JPAF paper (arXiv:2601.10025) — Jungian Personality Adaptation Framework: differentiation levels, decision rules, reinforcement-compensation, reflection mechanism
- LLMPTBench (NeurIPS 2025, OpenReview kVXePuKReA) — agentic frameworks exhibit exaggerated personality shifts under negative events
- BFI-Adapt (arXiv:2608.06485) — event-induced personality change benchmark, LLM agents shift indiscriminately and under-shift vs humans
- Takata et al. (2024, arXiv:2411.03252) — spontaneous emergence of agent individuality through social interactions
- Zeng et al. (ACL 2025, aclanthology.org/2025.findings-acl.1185) — dynamic personality in LLM agents, evolutionary modelling
- Li et al. (2024) — Cognition-Emotion-Growth architecture, feedback-loop personality development
