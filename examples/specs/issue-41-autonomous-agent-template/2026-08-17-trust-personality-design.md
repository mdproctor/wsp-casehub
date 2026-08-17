# Trust and Personality Design — Issue #45

## Context

Issue #45 wires the trust and personality systems into wacky-manor. All platform
APIs exist and all dependencies are already declared. The work is pure wiring —
implement one SPI, call existing utilities, and hook into the tick loop.

Dependencies: #43 (goal lifecycle — outcome tracking feeds trust signals),
#42 (memory stack — relationship memories feed trust scoring).

## Architecture

Five components, following the existing manor pattern (SPI impl + config wiring):

### 1. ManorTrustProvider

Implements `AgentTrustProvider` SPI (`currentTrustScore(agentId) → OptionalDouble`).

Computes trust from relationship memories stored by #42:
- Query `CaseMemoryStore` for relationship memories involving the agent
- Classify each memory as positive or negative based on content keywords
  (help, protect, warn → positive; lie, steal, betray, trick, trap → negative)
- Score = (positive × positiveWeight + negative × negativeWeight) / total,
  normalized to 0.0–1.0
- Default trust for unknown agents: 0.5

CDI `@ApplicationScoped` bean. Injected into `AgentExperienceService` for
trust-based retention, and called during observation building for trust context.

### 2. ManorDispositionRecorder

Records `BehavioralSignal` and disposition axis activations after each action.

Outcome-based mapping (D9) — only meaningful actions generate signals:

| ActionType | Outcome | BehavioralSignal | DispositionAxis |
|---|---|---|---|
| STEAL | any | SUCCESS | RISK_APPETITE |
| GIVE | success | SUCCESS | SOCIAL_ORIENTATION |
| PULL_ASIDE | success | SUCCESS | AUTONOMY |
| USE | success | SUCCESS | RISK_APPETITE |
| INTERACT | success | SUCCESS | SOCIAL_ORIENTATION |
| STEAL | failed | DECLINE | — (no axis, just behavioral) |
| MOVE, LOOK, WAIT | any | — (skip) | — (skip) |

Constructed in `runScenario()`, called in the post-action phase of
`runAutonomousTicks()` after action resolution.

### 3. ManorPersonalityEvolution

Wraps `DispositionEvolution.evaluate()`. Called periodically (every
`manor.disposition.evolution-check-interval` ticks) during the reflection
cascade in `ingest()`.

Flow:
1. Check if enough ticks have elapsed since last evolution check
2. Query `DispositionSignalStore.activationCounts()` for the agent
3. Build `EvolutionPending` from accumulated signals
4. Call `DispositionEvolution.evaluate(descriptor, pending)`
5. If `Evolved`: update the agent descriptor in `AgentRegistry`
6. Decay signal counts after evaluation

### 4. Personality-Weighted Retrieval

Not a new class. Inline call in `ScenarioOrchestrator.runAutonomousTicks()`:

```java
var memories = experienceService.recall(c.agentId(), recallLimit);
if (personalityWeightedRetrieval) {
    var weights = deriveWeights(c);  // from Eidos descriptor disposition
    memories = PersonalityWeightedRetrieval.reweight(memories, weights, Instant.now());
}
```

Derives `PersonalityWeights` from the character's Eidos disposition axes.

### 5. Trust-Based Retention

Extend the existing `purge()` call in `AgentExperienceService` reflection path.
After reflection fires, purge using `CbrRetentionPolicy` with `minTrustScore`
from `ManorTrustProvider`:

- Low-trust agent memories get deprioritized (lower salience score)
- Below `minTrustScore` threshold, memories are candidates for purge

## Hook Points in the Tick Loop

```
Pre-action:
  recall memories
    → reweight by personality (new)
    → build observation (include trust scores for co-located characters)
    → invoke LLM

Post-action:
  resolve action
    → record disposition signals (new)
    → ingest experience
        → reflection trigger
            → personality evolution check (new)
            → trust-weighted purge (new)
```

## Configuration

```yaml
manor.disposition.enabled: true
manor.disposition.evolution-check-interval: 5
manor.trust.enabled: true
manor.trust.positive-weight: 1.0
manor.trust.negative-weight: -2.0
manor.personality.weighted-retrieval: true
```

## New Classes

| Class | Package | Pattern |
|---|---|---|
| `ManorTrustProvider` | `io.casehub.examples.manor.agent` | CDI bean, implements `AgentTrustProvider` |
| `ManorDispositionRecorder` | `io.casehub.examples.manor.agent` | POJO, constructed in orchestrator |
| `ManorPersonalityEvolution` | `io.casehub.examples.manor.agent` | POJO, constructed in orchestrator |

## Modified Classes

| Class | Change |
|---|---|
| `ScenarioOrchestrator` | Wire new components, add config properties, add disposition recording and personality-weighted retrieval to tick loop |
| `AgentExperienceService` | Accept `ManorTrustProvider` and `ManorPersonalityEvolution`, wire into reflection/purge path |
| `ObservationBuilder` | Add trust score section for co-located characters |

## Testing

Unit tests for each new class:
- `ManorTrustProviderTest` — positive/negative memory scoring, default trust, edge cases
- `ManorDispositionRecorderTest` — action-type to signal mapping, skip rules
- `ManorPersonalityEvolutionTest` — evolution trigger, descriptor update, decay

## References

- `AgentTrustProvider` — `io.casehub.neocortex.memory.cbr.AgentTrustProvider`
- `BehavioralSignalStore` — `io.casehub.eidos.api.BehavioralSignalStore`
- `DispositionSignalStore` — `io.casehub.eidos.api.DispositionSignalStore`
- `DispositionEvolution` — `io.casehub.eidos.api.DispositionEvolution`
- `PersonalityWeightedRetrieval` — `io.casehub.neocortex.memory.personality.PersonalityWeightedRetrieval`
- `CbrRetentionPolicy` — `io.casehub.neocortex.memory.cbr.CbrRetentionPolicy`
- Epic #41 — issue #45 scope definition
- #42 commit d97905e — relationship memory wiring (trust input)
- #43 commit 933eb82 — goal revision strategy (outcome tracking pattern)
