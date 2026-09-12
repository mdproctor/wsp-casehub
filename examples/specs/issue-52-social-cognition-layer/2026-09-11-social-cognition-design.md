# Social Cognition Layer — Design Spec (Revised)

**Issue:** casehubio/examples#52
**Date:** 2026-09-11 (revised)
**Status:** Draft — revised after deep audit of neocortex/blocks capabilities

---

## Problem

Wacky-manor characters operate on a stateless observe → act loop. They
have no persistent beliefs about the world, no social drives shaping
behavior, no normative rules constraining actions, and flat episodic
memory with no structured knowledge graph.

Meanwhile, neocortex and blocks already have a rich cognitive
infrastructure — CognitiveProfile, PerspectivalResolver,
CognitiveDerivationEngine, ConversationBridge, consolidation phases,
mindmap with typed nodes, per-principal isolation with sharing — all
built, none consumed by an application.

**Wacky-manor should be the first real consumer of this cognitive stack.**
The work is primarily wiring, not building.

---

## Architecture: Three-Tier Memory + Cognitive Profile

### Three-Tier Memory Model

| Tier | What | Storage | Lifecycle |
|------|------|---------|-----------|
| **Tier 1 — Working memory** | Current observation, last action, current thinking | In-process (`CharacterState` + tick context) | Rebuilt every tick, never persisted |
| **Tier 2 — Episodic buffer** | Recent events not yet consolidated | In-process (`AgentExperienceService`) | Accumulates during awake gameplay, consumed during consolidation |
| **Tier 3 — Knowledge graph** | Consolidated knowledge: entities, relationships, beliefs, trust, judgments | Neocortex mindmap (per-character via `principalId`) | Written during consolidation (sleep), queried for cognitive profile |

Neocortex mindmap is never written to during the high-frequency tick
loop. Consolidation bridges Tier 2 → Tier 3 during sleep.

### Per-Character Isolation + Sharing

Already built in neocortex:
- `MindMapNode.principalId()` — ownership per node
- `MindMapNode.sharedWith()` — explicit sharing with other principals
- `PerspectivalResolver` + `PerspectivalMerge.merge(shared, overlay)` — per-character views on shared nodes

Conversation transfers knowledge between graphs:
- HC tells Penelope "poison is in the Ballroom" → Penelope's graph gets a BELIEF node with `confidenceOrigin: STATED`, confidence weighted by trust(HC)
- Penelope later finds poison in Kitchen → contradicts HC's claim → trust(HC) drops
- `ConversationBridge.process()` handles text → mindmap with LLM entity extraction + contradiction detection — already built

### Typed Nodes via TypeRegistry

Neocortex `TypeRegistry` supports dynamic type registration. SubgraphType
was migrated from enum to strings. Existing types: PERSON, PROJECT,
ORGANISATION, CONCEPT, RESEARCH_AREA, GENERAL.

**Register cognitive types** (neocortex#322 — XS):
BELIEF, INTENTION, PREDICTION, JUDGMENT, FEAR, DESIRE — parented
under CONCEPT, with trait interfaces following the Eventlike/Personable
pattern.

**Register game-world types** (wacky-manor):
ITEM, LOCATION, CHARACTER — via `typeRegistry.registerType()` at scenario
start.

Consolidation processes each type differently:
- BELIEF → revised on contradicting evidence
- INTENTION → evaluated for viability against current state
- FEAR → checked against current threats
- JUDGMENT → strengthened/weakened by new data
- Episodic events → abstracted into semantic knowledge via `CommunitySummaryPhase`

### Consolidation as a Game Mechanic

"Night falls on the mansion. Characters rest and reflect..."

**Existing consolidation phases:**
- `AccessFrequencyPhase` — strengthen frequently accessed nodes
- `MergeDetectionPhase` — deduplicate similar nodes
- `CommunitySummaryPhase` — abstract event clusters into higher-level nodes
- `CuriosityRefreshPhase` — curiosity-driven exploration

**Needed:** `consolidateNow(tenantId)` on `ConsolidationScheduler`
(neocortex#323 — XS) so the game mechanic can trigger consolidation
explicitly.

**New phase needed:** `ExperienceConsolidationPhase` — graduates worthy
events from Tier 2 episodic buffer → Tier 3 mindmap nodes. Adapts
`ConversationBridge` for `ExperienceEvent` input. Uses blocks scoring
(surprise, arousal, confidence) to decide what graduates vs gets pruned.

---

## What Neocortex Already Provides (Wire, Don't Build)

| Capability | Neocortex Component | What it does |
|-----------|---------------------|-------------|
| Per-character knowledge | `principalId` + `sharedWith` on MindMapNode | Isolation + explicit sharing |
| Character-specific views | `PerspectivalResolver` + `PerspectivalMerge` | HC sees world differently than Penelope |
| Entity knowledge queries | `CognitiveProfile` → `EntityKnowledge` | "What does this character know about X?" across 6 domains |
| Personality → cognition | `CognitiveDerivationEngine` | Eidos descriptor → trust formation rate, conflict interpretation, curiosity config, social cognition defaults |
| Text → knowledge graph | `ConversationBridge` + `MindMapExtractor` | Segments text, creates nodes, extracts entities/relationships, detects contradictions |
| Memory consolidation | `ConsolidationScheduler` + 4 phases | Strengthen, deduplicate, abstract, explore |
| Attention/salience | `TemporalFocus` + `AttentionItem` | Rank what the character should be thinking about by proximity, recency, affect |
| Affect model | PAD (pleasure/arousal/dominance) on every node | Emotional coloring of knowledge |
| Confidence tracking | `ConfidenceOrigin` (STATED/INFERRED/SPECULATED) with decay | Epistemic provenance |
| Temporal marking | `TemporalMark`, `validFrom`/`validUntil` | Knowledge has temporal validity windows |
| Edge vocabulary | `MindMapVocabulary` with per-type decay half-life | Relationship types with configurable forgetting |

---

## What Blocks Already Provides (Wire, Don't Build)

| Capability | Blocks Component | What it does |
|-----------|-----------------|-------------|
| Observation rendering | `CognitiveObservationSections` | Render goals, activity, experience, insights, relationships → ObservationSection |
| Affordance pipeline | `ObservationPipeline` + `PerceptionFilter` | Capability-tag-based observation filtering |
| Drive model | `DriveProfile`, `DriveConfig`, `DriveOrchestrator` | 4 SDT axes + `motivationalStateSection()` rendering |
| Drive → prompt | `DrivePromptSection`, `SocialPromptAssembler` | Render drives into prompts |
| Drive → goals | `DriveGoalFormationStrategy` | Map drives to emergent goals |
| Normative reasoning | `ConflictResolutionStrategy`, `NormDecision`, `PriorityResolution` etc. | Resolve conflicting norms |
| Memory scoring | `SurpriseScorer`, `ArousalScorer`, `ConfidenceScorer` | Score events for consolidation importance |
| Memory hygiene | `MemoryHygieneOrchestrator`, `RetentionConfig` | Automatic memory cleanup |
| Narrative | `NarrativeOrchestrator`, `NarrativeSynthesiser` | Narrative generation from social dynamics |
| Belief model | `Belief<T>`, `BeliefSet<T>`, `ConsistencyChecker` | Immutable belief sets with entrenchment-based revision |

---

## What Needs Building

### Neocortex (upstream — benefits all cognitive agents)

| Issue | What | Size |
|-------|------|------|
| #322 | Register cognitive node types + trait interfaces | S |
| #323 | `consolidateNow(tenantId)` public trigger | XS |
| *new* | Extend MindMapExtractor prompt with cognitive types | XS |
| *new* | `ExperienceConsolidationPhase` — Tier 2 → Tier 3 graduation | S |

### Blocks (upstream — benefits all cognitive agents)

| Issue | What | Size |
|-------|------|------|
| #260 | `CognitiveObservationSections` — beliefs, principles, trust, norms renderers | S |

### Wacky-Manor (app-specific)

| What | Size |
|------|------|
| **Refactor: Extract `CharacterCognition`** from ScenarioOrchestrator — per-character object owning cognitive state, querying all three tiers | L |
| **Refactor: ObservationBuilder → builder pattern** — growing parameter list, exchange path doesn't need full cognition | S |
| **Refactor: Config records** — group 30+ config properties into typed records | S |
| **Refactor: AgentExperienceService constructors** — telescoping 13-param constructors → builder/config record | S |
| **Wire `CognitiveDerivationEngine`** — Eidos descriptors → cognitive defaults (trust formation, social cognition, curiosity) | S |
| **Wire `ConversationBridge`** — dialogue events → mindmap knowledge extraction + contradiction detection | S |
| **Wire `CognitiveProfile`/`PerspectivalResolver`** — query mindmap for cognitive observation sections | M |
| **Wire consolidation** — sleep game mechanic triggering `consolidateNow()` per character | S |
| **Register game-world types** — ITEM, LOCATION, CHARACTER via TypeRegistry | XS |
| **`ManorNormFilter`** — context-filter norms by room, nearby characters, inventory | S |
| **`ManorTrustEvents`** — map ActionType → TrustEvent for relationship trust recording | XS |
| **Character descriptor extensions** — social config in Eidos extensionData for 5 core characters | S |
| **Integration + LLM eval tests** | M |

---

## Cognitive Profile Flow (Per Character Per Tick)

```
AWAKE (per tick):
  1. Tier 1: Build working memory (current observation, world state)
  2. Tier 2: Drain episodic buffer (recent events)
  3. Tier 3: Query mindmap via CognitiveProfile:
     - beliefs (BELIEF nodes for this principal)
     - trust (relationship edges to nearby characters)
     - principles (high-entrenchment identity nodes)
     - active norms (filtered by context via ManorNormFilter)
     - fears/intentions (if any)
  4. Render via CognitiveObservationSections (existing + new #260)
  5. Build observation (ObservationBuilder with all tiers)
  6. Invoke LLM
  7. Process response: dialogue, actions
  8. Record trust events → relationship edges on mindmap
  9. Ingest experience → Tier 2 episodic buffer
  10. ConversationBridge: dialogue text → entity extraction → shared nodes

SLEEP (consolidation — between acts):
  "Night falls on the mansion..."
  Per character: consolidateNow(principalId)
  - ExperienceConsolidationPhase: Tier 2 → Tier 3 (scored by surprise/arousal)
  - AccessFrequencyPhase: strengthen important nodes
  - MergeDetectionPhase: deduplicate
  - CommunitySummaryPhase: abstract events into beliefs/judgments
  - CuriosityRefreshPhase: exploration
  Characters wake with evolved understanding.
```

---

## Design Decisions

### D1: Integration approach — Wire existing platform (revised from Hybrid)

Wire neocortex CognitiveProfile/PerspectivalResolver/ConversationBridge/
consolidation and blocks scoring/normative/drives/observation rendering.
Build only app-specific adapters in wacky-manor.

### D2: Configuration source — Eidos descriptors + CognitiveDerivationEngine

Social cognition defaults derived from personality via
`CognitiveDerivationEngine`. Per-character overrides in Eidos
`extensionData.social`. Engine derives trust formation rate, conflict
interpretation, curiosity config from Jungian function stack and
disposition.

### D3: Influence mode — Prompt-visible via CognitiveObservationSections

All cognitive state rendered into observation sections the LLM sees.
Uses existing + new (#260) `CognitiveObservationSections` methods.

### D4: Phasing — Upstream first, then wacky-manor

Neocortex #322, #323 + blocks #260 first (small, unblocking).
Then wacky-manor refactoring + wiring.

### D5: Memory architecture — Three-tier with mindmap-native cognition

Working memory (in-process) → episodic buffer (in-process) → knowledge
graph (neocortex mindmap). Consolidation bridges Tier 2 → 3 during
sleep. Beliefs, trust, principles, judgments are mindmap queries, not
separate stores.

### D7: Storage model — Unified mindmap with Thing trait facades

All consolidated cognitive state on the mindmap. Per-concept access via
Thing trait projections: `node.as(Belieflike.class)`. Collection queries
via CognitiveProfile filtered by type. No separate wrapper classes —
the Thing API IS the facade pattern.

Validated via multi-agent debate (unified vs separate stores). Soar/ACT-R
research supports unified long-term storage with separate working buffers
— which is the three-tier model. The separate advocate's strongest
contribution (typed per-concept APIs) is incorporated via Thing traits.

### D6: Norms vs constraints — Principles + contextual norms (no duplication)

Eidos constraints stay in system prompt (identity: "you ARE this way").
Principles copied to observation (active guidance: "remember, you follow
these"). Norms are situation-specific guidance filtered by context — a
different concept from identity-level constraints.

---

## What's NOT in Scope

- Goal/plan system changes — keep existing ManorGoal*/ManorPlan*
- RAG, CBR, speech, PatternSpec YAML
- Mechanical norm enforcement — norms are advisory
- Dynamic drives — static per scenario for now

---

## Upstream Dependencies

| Repo | Issue | What | Blocks wacky-manor? |
|------|-------|------|-------------------|
| neocortex | #322 | Cognitive node types + traits | Partially — wiring can start without, typing improves consolidation |
| neocortex | #323 | `consolidateNow()` trigger | Yes — sleep mechanic needs this |
| blocks | #260 | Cognitive observation section renderers | Partially — can use raw ObservationSection.items() as fallback |

None are hard blockers for starting the wacky-manor refactoring. The
upstream work can proceed in parallel.

---

## References

- `ScenarioOrchestrator.java:241-434` — autonomous tick loop
- `ObservationBuilder.java:1-88` — observation rendering
- `CognitiveProfile.java` (neocortex cognitive-index) — entity knowledge queries
- `PerspectivalResolver.java` (neocortex mindmap-intelligence) — per-character views
- `CognitiveDerivationEngine.java` (neocortex cognitive-index) — personality → cognition
- `ConversationBridge.java` (neocortex mindmap-intelligence) — text → mindmap pipeline
- `ConsolidationScheduler.java` (neocortex mindmap-intelligence) — consolidation orchestration
- `CognitiveObservationSections.java` (blocks) — observation section rendering
- `DriveProfile.java` (blocks) — drive model
- `ConflictResolutionStrategy.java` (blocks) — normative resolution
- `SurpriseScorer.java`, `ArousalScorer.java` (blocks) — memory scoring
- neocortex#322, neocortex#323, blocks#260 — upstream issues
- D1–D6 in `decisions.md` — captured design decisions
- GE-20260816-6635e1 — ChannelObserver bridging pattern
