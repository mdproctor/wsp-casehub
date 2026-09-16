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

Wire the ledger's Bayesian Beta trust model (`TrustScoreComputer`) into the consolidation pipeline. Gameplay actions create in-memory ledger entries (via `PlainLedgerEntry`); witnesses and affected characters create attestations. During consolidation, a new `TrustConsolidationPhase` queries the ledger, computes per-relationship trust scores, and writes them to person-entity overlay nodes. Observation rendering reads trust scores directly from overlay node properties during `CharacterCognition.renderCognitiveSections()`.

All framework code goes in the platform (blocks-core, blocks-agentic-yaml, neocortex). Wacky-manor provides only YAML configuration mapping action types to trust verdicts.

### Ledger vs Episodic Buffer (D14)

Issue #65 references extracting trust events from the "episodic buffer" (Tier 2 experience store). The design uses ledger entries instead because:

1. **Structured attestation model** — Trust events map naturally to the ledger's `LedgerEntry` + `LedgerAttestation` model with verdicts, confidence, and capability tags. The episodic buffer (`CaseMemoryStore`) stores unstructured experience text.
2. **Existing trust infrastructure** — `TrustScoreComputer` and `TrustScoreCalculator` already operate on ledger entries and attestations. Using the episodic buffer would require a parallel trust computation pipeline.
3. **Append-only auditability** — The ledger provides Merkle chain integrity for trust decisions. Trust history should be tamper-evident.
4. **Temporal scoring** — The `DecayFunction` operates on attestation timestamps already present on `LedgerAttestation.occurredAt`.

The episodic buffer remains the source for Tier 2 → Tier 3 experience graduation via `ExperienceConsolidationPhase`. Trust events flow through the ledger, not the episodic buffer.

## Architecture

### Module Split (D2)

The trust framework spans four modules, following existing architectural boundaries:

| Module | What | Why here |
|--------|------|----------|
| **blocks-core** | Pure computation: `TrustEvolutionConfig` (record), `OverlayTrustPropertyModel` (constants). Zero CDI. | blocks-core charter: framework-neutral POJOs |
| **blocks-agentic-yaml** | `TrustEvolutionConfigSpec` (YAML spec type). Follows existing pattern (`DriveConfigSpec`, `RetentionConfigSpec`). | agentic-yaml owns all YAML-backed cognitive specs |
| **blocks** | CDI wiring: `TrustEventRecorder` (CDI observer), `TrustRelevantAction` (CDI event type). Depends on quarkus-arc. | blocks owns CDI cross-cutting concerns |
| **neocortex-mindmap-intelligence** | `TrustConsolidationPhase` (implements `ConsolidationPhase`). Runs during consolidation, uses `TrustScoreComputer`, writes to overlay nodes. | All consolidation phases live here |

### New Dependencies

- neocortex-mindmap-intelligence gains `casehub-ledger-api` (for `LedgerEntryRepository`, `LedgerEntry`, `LedgerAttestation`)
- neocortex-mindmap-intelligence gains `casehub-ledger-core` (for `TrustScoreComputer`, `DecayFunction`)
- neocortex-mindmap-intelligence gains `casehub-blocks-core` (for `TrustEvolutionConfig`, `OverlayTrustPropertyModel`)

### Relationship to Existing Trust System

**`TrustScoreAgentTrustProvider`** (engine module) bridges `AgentTrustProvider` to `TrustScoreSource` for global trust scores. It serves CBR case weighting (`TrustWeightedCbrCaseMemoryStore`), retention purging (`TrustRetentionPurger`), and planning (`FsiPlanAdapter`). These consumers need a single global trust score per agent — not per-relationship trust.

This spec introduces **per-relationship trust** (observer→subject) via overlay nodes. These serve observation rendering — each character's subjective view of another character's trustworthiness.

The two systems are complementary:
- **Global trust** (`TrustScoreSource` → `AgentTrustProvider`): How trustworthy is agent X across all observers? Used by platform infrastructure.
- **Per-relationship trust** (overlay nodes → observation rendering): How much does observer A trust subject B? Used by cognitive observation pipeline.

`AgentTrustProvider` keeps its current signature `currentTrustScore(String agentId)` — it serves global trust consumers that have no observer context. Per-relationship trust bypasses this SPI entirely, reading directly from overlay node properties. `TrustScoreAgentTrustProvider` is `@DefaultBean` and remains unchanged.

**`ManorTrustEvents`** (wacky-manor) is the existing weight-based trust model: STEAL=−0.4, GIVE=+0.15, with personality modifiers (`trustFormationRate`, `conflictInterpretation`). `CharacterCognition.recordTrustEvent()` calls it during action processing.

This spec supersedes `ManorTrustEvents`. The CDI event mechanism (`TrustRelevantAction` → `TrustEventRecorder` → ledger) replaces the weight-based model with a Bayesian attestation model. `CharacterCognition.recordTrustEvent()` becomes the call site for firing `TrustRelevantAction` CDI events instead of calling `ManorTrustEvents.weightFor()`.

**Personality modifiers**: The existing `ManorTrustEvents` applies `trustFormationRate` and `conflictInterpretation` to weight calculations. The new model does not apply personality modifiers to attestation confidence — all characters evaluate the same action identically. This is a deliberate scope decision: personality-driven trust interpretation is captured as a follow-up issue where `CognitiveDerivationEngine` modifies effective confidence weights during consolidation scoring based on character personality. Applying personality during consolidation (not at recording time) keeps ledger records personality-neutral, allowing personality changes to retroactively affect trust computation.

### Data Flow

```
TICK PROCESSING (per action):
  Character A performs action (STEAL, GIVE, INTERACT, ...)
    → CharacterCognition.recordTrustEvent(targetId, action)
      → fires TrustRelevantAction CDI event (D10)
        → TrustEventRecorder observes (D7):
          1. Maps actionType to verdict+confidence via TrustEvolutionConfig
             (unmapped types silently ignored — no entry created)
          2. Creates PlainLedgerEntry (actorId=A, entryType=EVENT,
             subjectId=UUID.nameUUIDFromBytes(targetId.getBytes()))
             → saved to InMemoryLedgerEntryRepository
          3. Creates LedgerAttestation per affected/witnessing character:
             - attestorId = observer character ID
             - verdict = SOUND (positive) or FLAGGED (negative) per config
             - confidence = from config (direct target vs witness)
             → saved to same in-memory repo

CONSOLIDATION ("sleep"):
  ConsolidationScheduler.consolidateNow(tenantId)
    → phases run in @Priority order:
      ... AccessFrequency(10) → Experience(15) → MergeDetection(20) →
      → TrustConsolidationPhase(22):                                    ← NEW
        For each overlay node in "people" subgraph:
          1. Read agentId property (observer) and shared node's agentId (target)
          2. Query findByActorId(targetId, ...) for target's entries
          3. For each entry: findAttestationsByEntryId(entry.id, tenancyId)
          4. Filter attestations: keep only attestorId == observerId
          5. Build Map<UUID, List<LedgerAttestation>> from filtered results
          6. Construct verdict-aware DecayFunction from config (D1)
          7. Call TrustScoreComputer(decay).compute(entries, map, now)
             → ActorScore
          8. Update overlay node: trust-score, trust-alpha, trust-beta
          9. If |trust-score − trust-last-rendered| > threshold (D8):
             update trust-last-rendered
      → SchemaDiscovery(25) → CommunitySummary(30) → CuriosityRefresh(40)

OBSERVATION RENDERING (next tick):
  CharacterCognition.renderCognitiveSections(...)
    → queries "people" subgraph overlay nodes for this character
    → reads trust-score, trust-alpha, trust-beta from overlay properties
    → maps to TrustLevel using config thresholds
    → constructs TrustSummary(subjectName, level, reason)
    → passes to CognitiveObservationSections.trustSection()
    → returns ObservationSection for the observation prompt
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

Constants for the trust properties on overlay nodes:

```java
public final class OverlayTrustPropertyModel {
    public static final String TRUST_SCORE = "trust-score";
    public static final String TRUST_ALPHA = "trust-alpha";
    public static final String TRUST_BETA = "trust-beta";
    public static final String TRUST_LAST_RENDERED = "trust-last-rendered";
}
```

### blocks-agentic-yaml

#### TrustEvolutionConfigSpec

YAML spec type in `agentic-yaml/src/main/java/.../spec/cognition/`, following the existing cognitive spec pattern (`DriveConfigSpec`, `RetentionConfigSpec`):

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

Fired by `CharacterCognition.recordTrustEvent()` after action resolution:

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

Observes `TrustRelevantAction` events asynchronously. Maps action type to verdict using `TrustEvolutionConfig`, creates `PlainLedgerEntry` + `LedgerAttestation` records via `LedgerEntryRepository`:

- Creates one `PlainLedgerEntry` per action with `actorId = action actor`, `entryType = EVENT`, `subjectId = UUID.nameUUIDFromBytes(targetId.getBytes())` — deterministic UUID from target character ID for queryability via `findBySubjectId()`
- Creates one `LedgerAttestation` per affected character (target + involved parties) with verdict from config, confidence from config
- Creates one `LedgerAttestation` per witness with same verdict, `witnessConfidence` from config
- Unmapped action types are silently ignored (no entry created)

`PlainLedgerEntry` (from ADR-0016) is the domain-agnostic concrete `LedgerEntry` subclass designed for generic EVENT entries. It has no subclass-specific fields, zero-column join table, and works with both in-memory and JPA stores without additional entity mapping or Flyway migrations.

### neocortex-mindmap-intelligence

#### TrustConsolidationPhase

Implements `ConsolidationPhase`. `@Priority(22)` — after `MergeDetectionPhase`(20), before `SchemaDiscoveryPhase`(25).

Algorithm:
1. Find the "people" subgraph via `MindMapStore.listSubgraphs()`
2. Query overlay nodes (trait = "overlay") in that subgraph
3. For each overlay node:
   a. Read `OverlayRef.AGENT_ID` (observer) and the shared node's `agentId` property (target)
   b. Query `LedgerEntryRepository.findByActorId(targetId, Instant.EPOCH, now, tenancyId)` for the target's ledger entries
   c. For each entry, query `findAttestationsByEntryId(entry.id, tenancyId)` and filter by `attestorId == observerId`
   d. Build `Map<UUID, List<LedgerAttestation>>` keyed by `entry.id` from filtered results
   e. Construct verdict-aware `DecayFunction` from config:
      ```java
      DecayFunction decay = (ageInDays, verdict) -> {
          int effectiveHalfLife = (verdict == FLAGGED || verdict == CHALLENGED)
              ? (int)(config.scoring().decayHalfLifeDays()
                      * config.scoring().negativeDecayMultiplier())
              : config.scoring().decayHalfLifeDays();
          return Math.pow(2.0, -(double)ageInDays / effectiveHalfLife);
      };
      ```
   f. Call `new TrustScoreComputer(decay).compute(entries, attestationMap, Instant.now())` → `ActorScore`
   g. Update overlay node via `MindMapStore.updateNode()`:
      - `trust-score` = ActorScore.trustScore()
      - `trust-alpha` = ActorScore.alpha()
      - `trust-beta` = ActorScore.beta()
   h. If `|trustScore - previousTrustLastRendered| > significantChangeThreshold`:
      - Update `trust-last-rendered` property

### wacky-manor (configuration only)

#### trust-evolution.yaml

Separate YAML file at `META-INF/eidos/trust-evolution.yaml`. Not merged into `social-config.yaml`, which uses a character-keyed top-level structure (`Map<String, Map<String, Object>>`). Trust evolution config is global — not per-character.

```yaml
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
  - type: GIVE
    verdict: SOUND
    confidence: 0.7
    witness-confidence: 0.3
  - type: INTERACT
    verdict: SOUND
    confidence: 0.3
    witness-confidence: 0.1
  - type: PULL_ASIDE
    verdict: SOUND
    confidence: 0.5
    witness-confidence: 0.2
  - type: USE
    verdict: FLAGGED
    confidence: 0.4
    witness-confidence: 0.2
```

All action types map to existing `ActionType` enum values (STEAL, GIVE, INTERACT, PULL_ASIDE, USE). Unmapped types (MOVE, LOOK, WAIT, TAKE) are not trust-relevant and produce no ledger entries.

#### ManorTrustEvolutionConfigLoader

New loader that parses `trust-evolution.yaml` into a `TrustEvolutionConfig` record. Separate from `ManorSocialConfigLoader` which handles per-character social config with character-keyed structure.

#### ScenarioOrchestrator wiring

`CharacterCognition.recordTrustEvent()` fires the CDI event:
```java
trustEvent.fire(new TrustRelevantAction(
    actorId, targetId, action.name(), result, witnessIds, tenantId));
```

The existing `ManorTrustEvents` weight-based model is superseded. `CharacterCognition.recordTrustEvent()` replaces its `ManorTrustEvents.weightFor()` call with the CDI event fire above.

## Observation Rendering (D8)

Trust changes integrate with the existing `TrustLevel` enum and `TrustSummary` record in blocks-core.

**Producer flow** (in `CharacterCognition.renderCognitiveSections()`):
1. Query the "people" subgraph for overlay nodes where `agentId` matches this character (observer)
2. For each overlay node with a `trust-score` property:
   a. Read `trust-score`, `trust-alpha`, `trust-beta`
   b. Resolve subject name from the shared node
   c. Map trust-score to `TrustLevel` using config thresholds:
      - `trust-score ≥ 0.7` → `TrustLevel.HIGH` → "You've come to rely on {name}"
      - `0.4 ≤ trust-score < 0.7` → `TrustLevel.MODERATE` → "You have mixed feelings about {name}"
      - `trust-score < 0.4` → `TrustLevel.LOW` → "Something about {name} makes you uneasy"
   d. Build reason string from evidence strength (alpha + beta):
      - `alpha + beta > 10` → "You feel quite certain about this"
      - `alpha + beta < 4` → "You're still forming an opinion"
   e. Construct `TrustSummary(subjectName, level, reason)`
3. Pass `List<TrustSummary>` to `CognitiveObservationSections.trustSection()`
4. Add the resulting `ObservationSection` to the observation prompt

## Persistence (D9)

All framework classes depend on `LedgerEntryRepository` (SPI), not implementations. `PlainLedgerEntry` (from ADR-0016) is the concrete subclass — domain-agnostic, zero-column join table, works with both in-memory and JPA stores. No new `LedgerEntry` subclass is needed.

Wacky-manor uses `InMemoryLedgerEntryRepository` (`@Alternative @Priority(1)`) — suitable for single game sessions. Production cognitive agents configure JPA-backed stores for durable trust history. The switch is a Quarkus `selected-alternatives` config change, not a code change.

## Trust Granularity (D11)

Trust is per-actor-pair (observer→subject), not per-capability. A character's trust in another character is a single scalar. The attestation `capabilityTag` defaults to `"*"`. Future enhancement: use `capabilityTag` for trust dimensions (honesty, reliability, scheming) without redesigning the model.

## Testing

### Unit tests (blocks-core)
- `TrustEvolutionConfig` construction and validation
- `OverlayTrustPropertyModel` constants

### Unit tests (blocks-agentic-yaml)
- `TrustEvolutionConfigSpec` YAML round-trip parsing

### Unit tests (blocks)
- `TrustEventRecorder`: action type → correct `PlainLedgerEntry` + `LedgerAttestation` creation, witness attestations at reduced confidence, unmapped action types ignored
- `TrustRelevantAction` record construction

### Unit tests (neocortex-mindmap-intelligence)
- `TrustConsolidationPhase`: overlay nodes updated with correct trust-score/alpha/beta, per-relationship filtering produces different scores for different observers, threshold gating suppresses small changes, empty attestation history produces 0.5 (Bayesian Beta prior), `DecayFunction` applies `negativeDecayMultiplier` for FLAGGED/CHALLENGED verdicts, `Map<UUID, List<LedgerAttestation>>` built correctly from filtered attestations

### Integration tests (wacky-manor)
- End-to-end: action → CDI event → ledger entry → consolidation → overlay node trust-score updated
- Observation rendering reads correct trust from overlay nodes after consolidation
- Multiple consolidation cycles with temporal decay shift scores
- YAML config round-trip: trust-evolution.yaml parsed correctly
- `ManorTrustEvents` superseded: verify CDI event path produces equivalent trust changes

### LLM eval tests (wacky-manor, -Pllm-eval profile)
- Trust change surfaces in observation text after consolidation
- Character behaviour adapts to evolved trust (qualitative)

## Follow-up Issues

- **Remove `ManorTrustEvents`**: After trust evolution is validated, remove the superseded weight-based trust model and its callers.
- **Personality-driven trust interpretation**: `CognitiveDerivationEngine` modifies effective confidence weights during consolidation scoring based on character personality (`trustFormationRate`, `conflictInterpretation`). Applying personality during consolidation keeps ledger records personality-neutral.
- **Per-capability trust dimensions**: Use `capabilityTag` for dimensions like honesty, reliability, scheming.
- **YAML DSL migration**: Port all of wacky-manor's social-config.yaml to blocks' agentic-yaml pipeline (drives, norms, beliefs, relationships, trust). Validates the DSL works end-to-end for cognitive agents.
- **Action model extension**: Add BETRAY, LIE, HELP, PROTECT to `ActionType` enum and wire into action resolution pipeline for richer trust events.

## References

- `io.casehub.ledger.core.trust.TrustScoreComputer` — Bayesian Beta trust computation
- `io.casehub.ledger.core.trust.TrustScoreCalculator` — four-pass algorithm wrapper
- `io.casehub.ledger.core.trust.DecayFunction` — temporal decay SPI
- `io.casehub.ledger.memory.InMemoryLedgerEntryRepository` — in-memory ledger stores
- `io.casehub.ledger.api.spi.LedgerEntryRepository` — persistence SPI
- `io.casehub.ledger.api.model.LedgerEntry` — ledger entry model (abstract)
- `io.casehub.ledger.runtime.model.PlainLedgerEntry` — domain-agnostic concrete subclass (ADR-0016)
- `io.casehub.ledger.api.model.LedgerAttestation` — attestation model
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationPhase` — consolidation SPI
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ExperienceConsolidationPhase` — @Priority(15)
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationScheduler` — consolidation orchestration
- `io.casehub.neocortex.mindmap.MindMapStore` — mindmap storage API
- `io.casehub.neocortex.mindmap.NodeUpdate` — node update API
- `io.casehub.neocortex.mindmap.OverlayRef` — overlay node linking
- `io.casehub.neocortex.memory.cbr.AgentTrustProvider` — trust SPI (global, not per-relationship)
- `io.casehub.engine.internal.worker.TrustScoreAgentTrustProvider` — existing global trust bridge (unchanged)
- `io.casehub.ledger.api.spi.TrustScoreSource` — global trust score source (unchanged)
- `io.casehub.blocks.summarisation.observation.affordance.TrustLevel` — trust level enum
- `io.casehub.blocks.summarisation.observation.affordance.TrustSummary` — trust observation record
- `io.casehub.blocks.summarisation.observation.affordance.CognitiveObservationSections` — observation rendering utility
- `io.casehub.blocks.agentic.yaml.spec.cognition.DriveConfigSpec` — existing YAML spec pattern
- `io.casehub.examples.manor.agent.ManorCognitiveSeeder` — person-entity overlay node creation
- `io.casehub.examples.manor.agent.ManorTrustEvents` — existing weight-based trust (superseded)
- `io.casehub.examples.manor.agent.CharacterCognition` — cognitive facade with recordTrustEvent()
- `io.casehub.examples.manor.agent.SocialConfig` — existing social config record
- `io.casehub.examples.manor.agent.ManorSocialConfigLoader` — per-character YAML loader
- `io.casehub.examples.manor.model.ActionType` — action type enum (MOVE, INTERACT, TAKE, GIVE, USE, LOOK, WAIT, STEAL, PULL_ASIDE)
- `io.casehub.neocortex.cognitive.index.SocialCognitionDefaults` — trustFormationRate, conflictInterpretation
- casehubio/examples#45 — trust and personality wiring (landed SPIs)
- casehubio/examples#61 — person-entity node seeding (overlay structure)
- casehubio/examples#52 — social cognition layer (three-tier memory model)
- casehubio/ledger#114 — PlainLedgerEntry (ADR-0016)
- GE-20260429-42fb02 — Bayesian Beta trust returns 0.5 for no evidence
- specs/issue-52-social-cognition-layer/2026-09-11-social-cognition-design.md
- specs/issue-58-content-scorer-refactoring/2026-09-15-person-entity-seeding-design.md
- specs/issue-41-autonomous-agent-template/2026-08-17-trust-personality-design.md
