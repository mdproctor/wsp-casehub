# CognitiveProfile Queries — Design Spec

**Issue:** casehubio/examples#54
**Epic:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Date:** 2026-09-14
**Status:** Draft

---

## Problem

CharacterCognition renders cognitive sections from static Java maps (SocialConfig). Beliefs, norms, and drives are hardcoded per character — they don't evolve through experience, conversation, or consolidation. The platform's cognitive query infrastructure (CognitiveProfile, TemporalFocus, CognitionCore, CognitiveObservationSections) is built but unwired. Characters lack dynamic cognition.

---

## Architecture

### CognitionCore Inside CharacterCognition

CharacterCognition becomes a thin adapter over CognitionCore (blocks#261). CognitionCore owns standard cognitive orchestration — mood, drives, narrative, prompt section rendering. CharacterCognition adds game-world context via a ManorContextStrategy that implements the future CognitionContextStrategy SPI contract (blocks#282) locally.

```
ScenarioOrchestrator
  └── CharacterCognition (per character)
        ├── CognitionCore — tick orchestration, mood, drives, prompt sections
        ├── ManorContextStrategy — game-world context adapter
        │     ├── norm filtering by room/nearby/inventory
        │     ├── trust scoping by proximity
        │     ├── ActionType → InteractionSignal mapping
        │     └── attention budget sizing
        ├── CognitiveProfile — mindmap queries (Tier 3 read path)
        └── TemporalFocus — attention-based item ranking
```

When blocks#282 lands, ManorContextStrategy becomes the SPI implementation — zero throwaway work.

### Per-Tick Query Flow

For each acting character per tick:

1. **CognitionCore.tick(agentId, tenantId, descriptor, activeSubjects)** — runs mood decay, drive evaluation, per-subject user/mental model ticking. `activeSubjects` = nearby agent IDs from world state, bridging room occupancy to the cognitive system's awareness model.
2. **ManorContextStrategy resolves situation** — nearby agents from world state, arousal from CognitionCore mood, budget from heuristics
3. **CognitiveProfile queries per entity:**
   - `CognitiveProfile.resolve(query.withAsSeenBy(principalId))` for each nearby entity and self-knowledge — returns `Optional<EntityKnowledge>` (node + edges + memories + trajectory + unresolved refs)
   - `CognitiveProfile.compare(query, nearbyAgentPrincipalIds)` for multi-agent perspectival views — returns `Map<PrincipalId, EntityKnowledge>` showing how different agents perceive the same entity. Used for theory-of-mind rendering (full social comparison UI deferred to #60).
4. **EntityKnowledge → TemporalEntry conversion:**
   - Each EntityKnowledge.node → `new TemporalEntry(node.updatedAt(), new FromMindMap(node), tenantId, node.confidence())`
   - Each memory across all domains → `new TemporalEntry(memory.timestamp(), new FromMemory(memory), tenantId, null)`
   - Each EntityKnowledge.trajectory → collected into `Map<String, AffectTrajectory>` keyed by node.id()
   - All entries flattened into a single `List<TemporalEntry>`
5. **TemporalFocus.focus(entries, now, trajectories, config)** — ranks all entries by salience (recency + affect trajectory + volatility), returns sorted `List<AttentionItem>`
6. **Budget selection** — take top items per category within adaptive budget
7. **Thing trait projection → blocks type mapping** — see §Mapping table below
8. **Blocks#260 rendering** — `beliefsSection()`, `trustSection()`, `normsSection()`, `principlesSection()`, plus `CognitionCore.promptSections()` for motivation/mood
9. **Post-action: InteractionSignal dispatch** — after action resolution, `CognitionCore.recordInteraction(agentId, tenantId, targetId, description, response)` replaces `recordTrustEvent()`. This is intentionally broader — handles mood appraisal, user model, mental model BDI extraction, not just trust. ActionType maps to `InteractionSignal.CustomSignal(description, quality)`: STEAL/USE → NEGATIVE, GIVE/INTERACT → POSITIVE, PULL_ASIDE → NEUTRAL.
10. **ObservationBuilder.withCognitiveSections()** — unchanged interface

### Adaptive Attention via TemporalFocus

The query scope is an attention problem, not a filtering problem. TemporalFocus scores every cognitive node by:

- **Recency** — `1 / (1 + hoursSince)`, recent items score higher
- **Affect trajectory** — worsening affect boosts, improving dampens, high volatility boosts
- **Proximity** — approaching temporal events score higher

A situational budget adapts to context:

| Signal | Budget effect |
|--------|--------------|
| Nearby agent count | More trust queries needed, budget per relationship adjusts |
| Arousal (from mood) | High arousal → larger overall budget (hyper-alert) |
| Goal pressure | Active goals boost budget for goal-relevant beliefs |

TemporalFocus is stateless, synchronous, pure function — callable every tick with no overhead. Falls back to recency-only scoring when no AffectTrajectory data exists (early game).

Configuration via `TemporalFocusConfig` knobs: `proximityScale`, `worseningBoostCap`, `improvingDampenFactor`, `volatilityBoostCap`.

---

## Seeding — Static to Dynamic Transition

During scenario initialization (in `runScenario()`, before the tick loop), `ManorCognitiveSeeder` writes SocialConfig initial beliefs to the mindmap as COGNITIVE-typed nodes with the character's principalId. The seeder runs per-character in the character construction loop, after CharacterCognition creation. It receives `MindMapStore` via CDI injection (`@ApplicationScoped`, available in wacky-manor's Quarkus context).

ManorCognitiveSeeder:
1. Creates one MindMapNode per InitialBelief, with the character's principalId and Belieflike trait properties
2. Records the set of seeded node IDs and their `updatedAt()` timestamps for later revised-belief detection
3. Runs once per scenario — deterministic, in the pre-tick initialization phase, no re-seeding guard needed

After seeding, CognitiveProfile queries are the sole source of truth for beliefs and trust.

| Cognitive element | Source after seeding | Why |
|-------------------|---------------------|-----|
| **Beliefs** | Mindmap (CognitiveProfile) | Knowledge — evolves through experience and consolidation |
| **Trust** | Mindmap edges (dynamic) | Accumulated from interactions, modulated by personality |
| **Personality drives** | SocialConfig (rendered by CharacterCognition as "Your Drives") | Character personality — static per scenario |
| **Motivation drives** | CognitionCore DriveOrchestrator (rendered as "Motivational State" via promptSections()) | SDT-based intrinsic motivation — dynamic, evaluated per tick |
| **Norms** | SocialConfig → ManorContextStrategy filtering | Behavioral rules — static, context-filtered per tick |
| **Principles** | Eidos constraints (AgentDescriptor) | Identity — stable, from system prompt |

Personality drives (SocialConfig: scheming, curiosity, gallantry) and motivation drives (CognitionCore: CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY) are semantically distinct systems. Personality drives are character-defining traits rendered as "Your Drives". Motivation drives are SDT-based intrinsic motivation axes computed from interaction data, rendered as "Motivational State". No duplication — different section titles, different semantics. CognitionConfig.drivesEnabled = true enables the motivation system alongside static personality drives.

Beliefs seed to the mindmap because they're revisable knowledge ("Penelope is naive" can be contradicted by experience). Norms stay in config because they're behavioral rules that don't evolve through consolidation.

---

## Mapping: Thing Traits → Blocks Types

CharacterCognition maps neocortex mindmap nodes to blocks renderer input types inline. Four conversions:

| Thing trait | Blocks type | Details |
|-------------|-------------|---------|
| `node.as(Belieflike.class)` | `Belief<String>` | See §Belieflike mapping |
| MindMapEdge | `TrustSummary` | See §Edge trust mapping |
| SocialConfig.NormEntry | `SocialNorm` | See §Norm mapping |
| AgentConstraint | `Principle` | `description` → `text`, category from constraint metadata |

### Belieflike → Belief\<String\>

- **key**: `belieflike.subject().orElse(node.name())` — subject is the belief topic identifier; falls back to node name when absent (Belieflike.subject() returns `Optional<String>`, but Belief.key is `@NonNull`)
- **value**: `node.name()` — the belief content as rendered text. When `belieflike.status()` is present, appended as `" [" + status + "]"` to preserve evidence provenance (e.g., "Penelope is naive [confirmed]")
- **entrenchment**: `(int)(node.confidence().value() * 10)` — maps Confidence.value [0.0,1.0] to integer entrenchment [0,10]. Affects sort order in `beliefsSection()` (descending). Belieflike.basis() is not used for entrenchment — basis is provenance metadata, not conviction strength.

### Edge → TrustSummary

- **subjectName**: resolved from target node name via `mindMapStore.getNode(edge.targetNodeId(), tenantId).name()`
- **level**: thresholded from `edge.confidence().value()` (Confidence is a record with double value in [0,1]):
  - ≥ 0.7 → `TrustLevel.HIGH`
  - ≥ 0.4 → `TrustLevel.MODERATE`
  - ≥ 0.1 → `TrustLevel.LOW`
  - < 0.1 → `TrustLevel.UNKNOWN`
- **reason** (`@Nullable`): `edge.edgeType()` describes the relationship type (e.g., "trust", "suspicion"). Enriched by `edge.provenance()` when non-null: `edgeType + " — " + provenance`

### SocialConfig.NormEntry → SocialNorm

SocialNorm requires 9 non-null fields. For statically-configured norms from SocialConfig:

| SocialNorm field | Source | Value |
|-----------------|--------|-------|
| normId | deterministic | `agentId + ":" + normEntry.rule().hashCode()` |
| description | NormEntry.rule() | e.g., "Never help Penelope directly" |
| behavioralPattern | NormEntry.rule() | same as description (static norms are their own pattern) |
| adherenceRate | constant | `1.0` (pre-configured norms assumed followed) |
| observationCount | constant | `0` (pre-configured, not dynamically observed) |
| participatingAgents | agentId | `Set.of(agentId)` (personal behavioral norm) |
| firstObserved | scenario start | `Instant.now()` at initialization |
| lastObserved | scenario start | same as firstObserved |
| strength | constant | `NormStrength.ESTABLISHED` |

### Revised belief detection

ManorCognitiveSeeder records the initial seeded node IDs and their `updatedAt()` timestamps per character. On each tick, when mapping beliefs from CognitiveProfile results:

- If a belief node's `updatedAt()` differs from the timestamp recorded at seeding → key added to `revisedKeys`
- New nodes not in the seeded set are also marked as revised (beliefs accumulated through consolidation)
- The `revisedKeys` set is passed to `beliefsSection()` which renders `[REVISED]` prefix per belief

---

## What Needs Building

### New types

| Type | Package | Purpose |
|------|---------|---------|
| `ManorContextStrategy` | `manor.agent` | CognitionContextStrategy adapter — game-world context |
| `CognitiveBudget` | `manor.agent` | Adaptive budget record with situational heuristics |
| `ManorCognitiveSeeder` | `manor.agent` | Seeds SocialConfig initial beliefs to mindmap at scenario start |

### Modified types

| Type | Change |
|------|--------|
| `CharacterCognition` | Compose CognitionCore + CognitiveProfile + TemporalFocus; replace renderCognitiveSections() internals; replace recordTrustEvent() with CognitionCore.recordInteraction() |
| `ScenarioOrchestrator` | Inject CognitiveProfile + CDI orchestrators; construct single shared CognitionCore; call seeder at scenario start; pass AffectTrajectory to TemporalFocus |
| `ManorConfig` | No changes — TemporalFocusConfig uses hardcoded defaults in ManorContextStrategy (see below) |

#### CognitionCore construction

CognitionCore is a **single shared instance** constructed in ScenarioOrchestrator during initialization. The orchestrators it composes (MoodOrchestrator, DriveOrchestrator, UserModelOrchestrator, MentalModelOrchestrator) are `@ApplicationScoped` CDI beans that key internally by `agentId:tenantId` — one CognitionCore serves all characters.

```java
new CognitionCore(
    moodOrchestrator,          // CDI @ApplicationScoped
    driveOrchestrator,         // CDI @ApplicationScoped — SDT motivation
    userModelOrchestrator,     // CDI @ApplicationScoped — social modeling
    mentalModelOrchestrator,   // CDI @ApplicationScoped — BDI extraction
    null,                      // strategy — not needed for manor
    null,                      // narrative — not needed for manor
    null,                      // goals — managed by ManorGoalEvaluator
    null,                      // memoryHygiene — not needed for manor
    agentProvider,             // for mood appraisal LLM calls
    CognitionConfig.all().without("strategy", "narrative", "goals", "memoryHygiene")
)
```

#### TemporalFocusConfig defaults

TemporalFocusConfig knobs use hardcoded defaults in ManorContextStrategy — no ManorConfig section needed for the initial implementation. Values: `proximityScale=7.0`, `worseningBoostCap=0.5`, `improvingDampenFactor=0.7`, `volatilityBoostCap=0.3`. These can be promoted to ManorConfig if tuning reveals the need.

### Dependencies

Already on classpath from Phase A:
- `casehub-neocortex-cognitive-index` (CognitiveProfile, TemporalFocus, Thing traits)
- `casehub-neocortex-mindmap-intelligence` (ConsolidationScheduler, mindmap types)
- `casehub-blocks` (CognitionCore, CognitiveObservationSections, renderers)

No new dependencies needed.

---

## Testing

**Unit tests (no CDI):**
- ManorContextStrategyTest — budget sizing across situations
- CognitiveBudgetTest — budget arithmetic
- Thing trait → blocks type mapping tests

**Integration tests (`@QuarkusTest`):**
- CognitiveQueryIntegrationTest — CognitiveProfile injectable, seeded beliefs retrievable
- TemporalFocusIntegrationTest — ranking and budget filtering produce correct results
- CognitionCoreWiringTest — tick runs, promptSections() returns sections
- Extend SocialCognitionIntegrationTest with dynamic query assertions

**Existing test preservation:** All 419 tests pass unchanged.

---

## Upstream Dependencies

| Repo | Issue | Status | Impact |
|------|-------|--------|--------|
| neocortex | #322 | CLOSED | Cognitive node types + traits — enables typed queries |
| neocortex | #323 | CLOSED | consolidateNow — already wired in Phase A |
| blocks | #260 | CLOSED | Cognitive observation renderers — used for output |
| blocks | #261 | CLOSED | CognitionCore — composed inside CharacterCognition |
| blocks | #282 | OPEN | CognitionContextStrategy SPI — ManorContextStrategy implements contract locally until it lands |

No blockers — all critical upstream work has landed.

---

## What's NOT in Scope

- ConversationBridge wiring (#55) — separate issue
- Moving SocialConfig to Eidos extensionData (#56) — separate issue
- Context-based ManorNormFilter enhancement (#57) — separate but ManorContextStrategy provides the hook
- LLM eval tests (#59) — separate issue, after this lands
- SocialComparison integration (#60) — separate issue, after this lands
- ExperienceConsolidationPhase (neocortex#336) — upstream, not in this branch

---

## Design Decisions

- D1: CognitionCore adoption — hybrid with local strategy adapter (blocks#282 contract)
- D2: Mapping layer — adapter in CharacterCognition (~4 inline conversions)
- D3: Static-to-dynamic transition — seed beliefs on first tick, mindmap authoritative after
- D4: Query scope — adaptive attention via TemporalFocus + situational budget

Full decision records: `specs/issue-54-cognitive-profile-queries/decisions.md`

---

## References

- Phase A design spec: `specs/issue-52-social-cognition-layer/2026-09-11-social-cognition-design.md`
- Phase A decisions: `specs/issue-52-social-cognition-layer/decisions.md` (D1-D7)
- CognitiveProfile (neocortex cognitive-index) — resolve(), compare()
- TemporalFocus (neocortex memory) — focus(), AttentionItem, TemporalFocusConfig
- CognitionCore (blocks) — tick(), promptSections(), CognitionConfig
- CognitiveObservationSections (blocks) — beliefsSection(), trustSection(), normsSection(), principlesSection()
- Thing traits (neocortex mindmap-intelligence) — Belieflike, Intentionlike, Evaluative, Fearlike, Desirelike, Predictive
- blocks#282 — CognitionContextStrategy SPI proposal
- neocortex#336 — ExperienceConsolidationPhase (upstream, future)
- CognitiveIndexWalkthroughTest (neocortex examples) — canonical consumer pattern
