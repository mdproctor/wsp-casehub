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

1. **CognitionCore.tick()** — runs mood, drives, narrative orchestrators
2. **ManorContextStrategy resolves situation** — nearby agents from world state, arousal from CognitionCore mood, budget from heuristics
3. **CognitiveProfile.resolve()** — queries mindmap per nearby entity and self-knowledge, using `query.withAsSeenBy(principalId)` for perspectival views
4. **TemporalFocus.focus()** — ranks all cognitive entries by salience (recency + affect trajectory + volatility)
5. **Budget selection** — take top items per category within adaptive budget
6. **Thing trait projection → blocks type mapping** — `node.as(Belieflike.class)` → `Belief<String>`, edge weights → `TrustSummary`, etc.
7. **Blocks#260 rendering** — `beliefsSection()`, `trustSection()`, `normsSection()`, `principlesSection()`, plus `CognitionCore.promptSections()` for drives/mood
8. **ObservationBuilder.withCognitiveSections()** — unchanged interface

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

At scenario start, SocialConfig initial beliefs are written to the mindmap as COGNITIVE-typed nodes with the character's principalId. After seeding, CognitiveProfile queries are the sole source of truth for beliefs and trust.

| Cognitive element | Source after seeding | Why |
|-------------------|---------------------|-----|
| **Beliefs** | Mindmap (CognitiveProfile) | Knowledge — evolves through experience and consolidation |
| **Trust** | Mindmap edges (dynamic) | Accumulated from interactions, modulated by personality |
| **Drives** | SocialConfig → CognitionCore DriveOrchestrator | Personality trait — static per scenario |
| **Norms** | SocialConfig → ManorContextStrategy filtering | Behavioral rules — static, context-filtered per tick |
| **Principles** | Eidos constraints (AgentDescriptor) | Identity — stable, from system prompt |

Beliefs seed to the mindmap because they're revisable knowledge ("Penelope is naive" can be contradicted by experience). Drives and norms stay in config because they're personality/behavioral rules that don't evolve through consolidation.

---

## Mapping: Thing Traits → Blocks Types

CharacterCognition maps neocortex mindmap nodes to blocks renderer input types inline. ~4 conversions:

| Thing trait | Blocks type | Mapping |
|-------------|-------------|---------|
| `node.as(Belieflike.class)` | `Belief<String>` | subject → key, node.summary() → value |
| Edge trust weight | `TrustSummary` | target name + weight → TrustLevel (HIGH/MODERATE/LOW/UNKNOWN) + reason from edge label |
| Norm (from SocialConfig) | `SocialNorm` | rule text + priority → SocialNorm with status ESTABLISHED |
| Constraint (from Eidos) | `Principle` | description → text, optional category |

Revised beliefs tracked via node metadata — `revisedKeys` set passed to `beliefsSection()` for visual marking.

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
| `CharacterCognition` | Compose CognitionCore + CognitiveProfile + TemporalFocus; replace renderCognitiveSections() internals; replace recordTrustEvent() with InteractionSignal dispatch |
| `ScenarioOrchestrator` | Inject CognitiveProfile; construct CognitionCore per character; call seeder at scenario start; pass AffectTrajectory to TemporalFocus |
| `ManorConfig` | No changes — consolidation config already present |

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
