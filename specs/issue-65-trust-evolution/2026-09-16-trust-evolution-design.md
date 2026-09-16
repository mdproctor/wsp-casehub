# Trust Evolution from Experience — Design Spec

**Issue:** casehubio/examples#65
**Parent:** casehubio/examples#64 (Phase C: Dynamic cognitive evolution)
**Date:** 2026-09-16
**Branch:** issue-65-trust-evolution

---

## Problem

Person-entity overlay nodes (#61) carry initial PAD values but no trust dimension. The AgentTrustProvider SPI exists in neocortex-memory-api, but no implementation reads trust from the mindmap. Trust-relevant gameplay events (help, betrayal, theft) accumulate during tick processing but nothing converts them into per-relationship trust scores during consolidation.

The result: characters have static perceptions of each other. Penelope's view of the Hooded Claw never changes regardless of how many times he betrays her.

## Solution

Wire the ledger's Bayesian Beta trust model (`TrustScoreComputer`) into the consolidation pipeline. Gameplay actions create in-memory ledger entries; witnesses and affected characters create attestations. During consolidation, a new `TrustConsolidationPhase` queries the ledger, computes per-relationship trust scores, and writes them to person-entity overlay nodes. An `OverlayTrustProvider` reads the scores back for observation rendering.

All framework code goes in the platform (blocks-core, blocks, neocortex). Wacky-manor provides only YAML configuration mapping action types to trust verdicts.

## Architecture

### Three-Module Split (D2)

The trust framework spans three modules, following existing architectural boundaries:

| Module | What | Why here |
|--------|------|----------|
| **blocks-core** | Pure computation: `TrustEvolutionConfig`, `OverlayTrustPropertyModel`, `TrustEvolutionConfigSpec`. Zero CDI. | blocks-core charter: framework-neutral POJOs |
| **blocks** | CDI wiring: `TrustEventRecorder` (CDI observer), `TrustRelevantAction` (CDI event type). Depends on quarkus-arc. | blocks owns CDI cross-cutting concerns |
| **neocortex-mindmap-intelligence** | `TrustConsolidationPhase` (implements `ConsolidationPhase`). Runs during consolidation, uses blocks-core computation, writes to overlay nodes. | All consolidation phases live here |

### New Dependencies

- blocks-core gains `casehub-ledger-core` (provided scope — pure Java, no CDI)
- neocortex-mindmap-intelligence gains `casehub-blocks-core` (for trust computation helpers)

### Data Flow

```
TICK PROCESSING (per action):
  Character A performs action (STEAL, HELP, GIVE, ...)
    → Orchestrator fires TrustRelevantAction CDI event (D10)
      → TrustEventRecorder observes (D7):
        1. Creates LedgerEntry (actorId=A, entryType=EVENT)
           → saved to InMemoryLedgerEntryRepository
        2. Creates LedgerAttestation per affected/witnessing character:
           - attestorId = observer character ID
           - verdict = SOUND (positive) or FLAGGED (negative) per YAML config
           - confidence = from YAML config (direct target vs witness)
           → saved to same in-memory repo

CONSOLIDATION ("sleep"):
  ConsolidationScheduler.consolidateNow(tenantId)
    → phases run in @Priority order:
      ... AccessFrequency(10) → Experience(15) → MergeDetection(20) →
      → TrustConsolidationPhase(22):                                    ← NEW
        For each overlay node in "people" subgraph:
          1. Extract observer ID and target ID from overlay properties
          2. Query LedgerEntryRepository for target's entries
          3. Filter attestations to observer's attestorId (D3)
          4. Feed to TrustScoreComputer.compute() → ActorScore
          5. Write trust-score, trust-alpha, trust-beta to overlay node (D5)
          6. If |trust-score - trust-last-rendered| > threshold (D8):
             emit trust change for observation rendering
      → SchemaDiscovery(25) → CommunitySummary(30) → CuriosityRefresh(40)

OBSERVATION RENDERING (next tick):
  OverlayTrustProvider.currentTrustScore(agentId)
    → reads trust-score from overlay node
    → returns OptionalDouble

  CognitiveObservationSections.trustSection()
    → reads TrustSummary(subjectName, level, reason)
    → renders into observation text
```

## Components

### blocks-core (pure Java, zero CDI)

#### TrustEvolutionConfig

Configuration record for the trust evolution pipeline:

```java
public record TrustEvolutionConfig(
    List<TrustEventMapping> events,
    ScoringConfig scoring,
    ConsolidationConfig consolidation,
    LevelConfig levels
) {
    public record TrustEventMapping(
        String actionType,
        AttestationVerdict verdict,
        double confidence,
        double witnessConfidence
    ) {}

    public record ScoringConfig(
        int decayHalfLifeDays,
        double negativeDecayMultiplier
    ) {}

    public record ConsolidationConfig(
        String overlayProperty,
        double significantChangeThreshold
    ) {}

    public record LevelConfig(
        double high,
        double moderate,
        double low
    ) {}
}
```

#### OverlayTrustPropertyModel

Constants and helpers for the trust properties on overlay nodes:

```java
public final class OverlayTrustPropertyModel {
    public static final String TRUST_SCORE = "trust-score";
    public static final String TRUST_ALPHA = "trust-alpha";
    public static final String TRUST_BETA = "trust-beta";
    public static final String TRUST_LAST_RENDERED = "trust-last-rendered";
}
```

#### TrustEvolutionConfigSpec

YAML spec type in `blocks/agentic-yaml/spec/cognition/`, following the existing cognitive spec pattern (`DriveConfigSpec`, `RetentionConfigSpec`, etc.):

```java
public class TrustEvolutionConfigSpec {
    public List<TrustEventMappingSpec> events;
    public TrustScoringSpec scoring;
    public TrustConsolidationSpec consolidation;
    public TrustLevelSpec levels;
}
```

### blocks (CDI layer)

#### TrustRelevantAction (CDI event)

Fired by orchestrators after action resolution:

```java
public record TrustRelevantAction(
    String actorId,
    String targetId,
    String actionType,
    String actionResult,
    List<String> witnessIds,
    String tenantId
) {}
```

#### TrustEventRecorder (CDI observer)

Observes `TrustRelevantAction` events asynchronously. Maps action type to verdict using `TrustEvolutionConfig`, creates `LedgerEntry` + `LedgerAttestation` records via `LedgerEntryRepository`:

- Creates one `LedgerEntry` with `actorId = action actor`, `entryType = EVENT`
- Creates one `LedgerAttestation` per affected character with verdict from config, confidence from config
- Creates one `LedgerAttestation` per witness with same verdict, `witnessConfidence` from config
- Uses `subjectId` derived from the actor-pair to partition ledger entries per relationship

### neocortex-mindmap-intelligence

#### TrustConsolidationPhase

Implements `ConsolidationPhase`. `@Priority(22)` — after `MergeDetectionPhase`(20), before `SchemaDiscoveryPhase`(25).

Algorithm:
1. Find the "people" subgraph via `MindMapStore.listSubgraphs()`
2. Query overlay nodes (trait = "overlay") in that subgraph
3. For each overlay node:
   a. Read `OverlayRef.AGENT_ID` (observer) and the shared node's `agentId` property (target)
   b. Query `LedgerEntryRepository.findByActorId(targetId, ...)` for the target's ledger entries
   c. Query attestations, filter by `attestorId == observerId` (D3)
   d. Instantiate `TrustScoreComputer` with `DecayFunction` from config (D1)
   e. Call `compute(entries, filteredAttestations, Instant.now())` → `ActorScore`
   f. Update overlay node via `MindMapStore.updateNode()`:
      - `trust-score` = ActorScore.trustScore()
      - `trust-alpha` = ActorScore.alpha()
      - `trust-beta` = ActorScore.beta()
   g. If `|trustScore - previousTrustLastRendered| > significantChangeThreshold`:
      - Update `trust-last-rendered` property
      - Emit trust change signal for observation rendering

### wacky-manor (configuration only)

#### social-config.yaml extension

Add `trust-events` section to social-config.yaml:

```yaml
# Top-level (not per-character — shared across all characters)
trust-events:
  scoring:
    decay-half-life-days: 30
    negative-decay-multiplier: 1.5
  consolidation:
    overlay-property: trust-score
    significant-change-threshold: 0.15
  levels:
    high: 0.7
    moderate: 0.4
    low: 0.2
  events:
    - type: STEAL
      verdict: FLAGGED
      confidence: 0.9
      witness-confidence: 0.5
    - type: BETRAY
      verdict: FLAGGED
      confidence: 1.0
      witness-confidence: 0.7
    - type: LIE
      verdict: FLAGGED
      confidence: 0.8
      witness-confidence: 0.4
    - type: HELP
      verdict: SOUND
      confidence: 0.8
      witness-confidence: 0.4
    - type: GIVE
      verdict: SOUND
      confidence: 0.7
      witness-confidence: 0.3
    - type: PROTECT
      verdict: SOUND
      confidence: 0.9
      witness-confidence: 0.5
```

#### ManorSocialConfigLoader extension

Parse the `trust-events` section into a `TrustEvolutionConfig` record. The loader adds one field to `SocialConfig` (or a parallel config record) and one parse block — no new class needed.

#### ScenarioOrchestrator wiring

Replace direct trust event calls with:
```java
trustEvent.fire(new TrustRelevantAction(
    actorId, targetId, actionType, result, witnessIds, tenantId));
```

### OverlayTrustProvider

Implements `AgentTrustProvider` SPI. CDI `@ApplicationScoped` bean in blocks (CDI layer). Reads `trust-score` property from overlay nodes via `MindMapStore`:

```java
public OptionalDouble currentTrustScore(String agentId) {
    // Find overlay node for the querying principal → agentId relationship
    // Read trust-score property
    // Return OptionalDouble.of(score) or OptionalDouble.empty()
}
```

## Observation Rendering (D8)

Trust changes integrate with the existing `TrustLevel` enum and `TrustSummary` record in blocks-core:

- `trust-score ≥ 0.7` → `TrustLevel.HIGH` → "You've come to rely on {name}"
- `0.4 ≤ trust-score < 0.7` → `TrustLevel.MODERATE` → "You have mixed feelings about {name}"
- `trust-score < 0.4` → `TrustLevel.LOW` → "Something about {name} makes you uneasy"

Evidence strength from alpha/beta shapes the qualifier:
- `alpha + beta > 10` → "You feel quite certain about this"
- `alpha + beta < 4` → "You're still forming an opinion"

## Persistence (D9)

All framework classes depend on `LedgerEntryRepository` (SPI), not implementations. Wacky-manor uses `InMemoryLedgerEntryRepository` (`@Alternative @Priority(1)`) — suitable for single game sessions. Production cognitive agents configure JPA-backed stores for durable trust history. The switch is a Quarkus `selected-alternatives` config change, not a code change.

## Trust Granularity (D11)

Trust is per-actor-pair (observer→subject), not per-capability. A character's trust in another character is a single scalar. The attestation `capabilityTag` defaults to `"*"`. Future enhancement: use `capabilityTag` for trust dimensions (honesty, reliability, scheming) without redesigning the model.

## Testing

### Unit tests (blocks-core)
- `TrustEvolutionConfig` construction and validation
- `OverlayTrustPropertyModel` constants

### Unit tests (blocks)
- `TrustEventRecorder`: action type → correct LedgerEntry + LedgerAttestation creation, witness attestations at reduced confidence, unmapped action types ignored
- `TrustRelevantAction` record construction

### Unit tests (neocortex-mindmap-intelligence)
- `TrustConsolidationPhase`: overlay nodes updated with correct trust-score/alpha/beta, per-relationship filtering produces different scores for different observers, threshold gating suppresses small changes, empty attestation history produces 0.5 (Bayesian Beta prior)

### Integration tests (wacky-manor)
- End-to-end: action → ledger entry → consolidation → overlay node trust-score updated
- OverlayTrustProvider reads correct score after consolidation
- Multiple consolidation cycles with temporal decay shift scores
- YAML config round-trip: social-config.yaml parsed correctly

### LLM eval tests (wacky-manor, -Pllm-eval profile)
- Trust change surfaces in observation text after consolidation
- Character behaviour adapts to evolved trust (qualitative)

## Follow-up Issues

- **YAML DSL migration**: Port all of wacky-manor's social-config.yaml to blocks' agentic-yaml pipeline (drives, norms, beliefs, relationships, trust). Validates the DSL works end-to-end for cognitive agents.
- **Personality-driven trust interpretation**: CognitiveDerivationEngine modifies confidence weights based on character personality (trusting characters weigh positive signals higher).
- **Per-capability trust dimensions**: Use capabilityTag for dimensions like honesty, reliability, scheming.

## References

- `io.casehub.ledger.core.trust.TrustScoreComputer` — Bayesian Beta trust computation
- `io.casehub.ledger.core.trust.TrustScoreCalculator` — four-pass algorithm wrapper
- `io.casehub.ledger.core.trust.DecayFunction` — temporal decay SPI
- `io.casehub.ledger.memory.InMemoryLedgerEntryRepository` — in-memory ledger stores
- `io.casehub.ledger.api.spi.LedgerEntryRepository` — persistence SPI
- `io.casehub.ledger.api.model.LedgerEntry` — ledger entry model
- `io.casehub.ledger.api.model.LedgerAttestation` — attestation model
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationPhase` — consolidation SPI
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ExperienceConsolidationPhase` — @Priority(15)
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationScheduler` — consolidation orchestration
- `io.casehub.neocortex.mindmap.MindMapStore` — mindmap storage API
- `io.casehub.neocortex.mindmap.NodeUpdate` — node update API
- `io.casehub.neocortex.mindmap.OverlayRef` — overlay node linking
- `io.casehub.neocortex.memory.cbr.AgentTrustProvider` — trust SPI
- `io.casehub.blocks.summarisation.observation.affordance.TrustLevel` — trust level enum
- `io.casehub.blocks.summarisation.observation.affordance.TrustSummary` — trust observation record
- `io.casehub.examples.manor.agent.ManorCognitiveSeeder` — person-entity overlay node creation
- `io.casehub.examples.manor.agent.SocialConfig` — existing social config record
- `io.casehub.examples.manor.agent.ManorSocialConfigLoader` — YAML loader
- `io.casehub.examples.manor.agent.PerceptionTranslator` — perception rendering
- casehubio/examples#45 — trust and personality wiring (landed SPIs)
- casehubio/examples#61 — person-entity node seeding (overlay structure)
- casehubio/examples#52 — social cognition layer (three-tier memory model)
- GE-20260429-42fb02 — Bayesian Beta trust returns 0.5 for no evidence
- specs/issue-52-social-cognition-layer/2026-09-11-social-cognition-design.md
- specs/issue-58-content-scorer-refactoring/2026-09-15-person-entity-seeding-design.md
- specs/issue-41-autonomous-agent-template/2026-08-17-trust-personality-design.md
