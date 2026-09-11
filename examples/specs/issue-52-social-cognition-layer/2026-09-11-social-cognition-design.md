# Social Cognition Layer — Design Spec

**Issue:** casehubio/examples#52
**Date:** 2026-09-11
**Status:** Draft

---

## Problem

Wacky-manor characters currently operate on a stateless observe → act loop.
Each tick, a character drains observations, recalls flat memories, and invokes
the LLM with world state. Characters have no persistent beliefs about the
world or other characters, no social drives shaping their behavior, and no
normative rules constraining their actions.

The result: characters react to what they see but don't form opinions, hold
grudges, follow personal codes, or pursue social agendas. The Hooded Claw
doesn't actively scheme against Penelope unless the observation happens to
contain poison — he has no drive to seek opportunities. Penelope doesn't
trust Peter Perfect more than strangers. Characters don't maintain beliefs
that persist across ticks and influence future decisions.

Meanwhile, blocks has built a full social cognition framework — belief revision,
social drives, normative reasoning, trust scoring, and memory hygiene — that
provides exactly these capabilities as pluggable building blocks.

## Approach

**Hybrid integration** (D1): Replace the simple trust and disposition subsystems
with blocks equivalents. Layer beliefs, social drives, and normative reasoning
as new additive cognitive inputs. Keep the working goal/plan/reflection system
unchanged.

| Component | Current | After |
|-----------|---------|-------|
| Trust | `ManorTrustProvider` (+1/-2 global scoring) | blocks `trust` — per-relationship, decay, context-aware |
| Disposition | `ManorDispositionRecorder` + `ManorPersonalityEvolution` | blocks `memory` — surprise/arousal/confidence scoring + hygiene orchestration |
| Beliefs | *none* | **New** — `BeliefStore` per character, revised on contradicting evidence |
| Social drives | *none* | **New** — drives from Eidos descriptors, prompt-visible |
| Norms | *none* | **New** — per-character rules with conflict resolution |
| Goals/Plans | `ManorGoalFormation/Revision`, `ManorPlanFormation/Revision` | **Keep** unchanged |
| Observations | `ObservationBuilder` + `ManorWorldObservationProvider` | **Extend** — new cognitive sections for beliefs, drives, norms |
| Reflection | `ManorReflectionTrigger/Synthesizer` | **Keep** — feed reflection outputs into belief revision |
| Memory | `AgentExperienceService` + neocortex flat memory | **Keep** for Phase A — Phase B replaces with mindmap |

---

## Design Decisions

### 1. Configuration source: Eidos descriptors (D2)

All per-character social cognition config lives in the character's Eidos YAML
descriptor. Eidos already owns personality traits, goals, and capabilities —
social cognition is a natural extension.

New `social:` section in each character descriptor:

```yaml
social:
  drives:
    - type: scheming
      intensity: 0.9
      description: "Compelled to hatch elaborate plans against Penelope"
    - type: self-preservation
      intensity: 0.7
      description: "Avoids direct confrontation, prefers subterfuge"
  norms:
    - rule: "Never help Penelope directly"
      priority: 10
    - rule: "Maintain a veneer of charm in public"
      priority: 5
    - rule: "Protect personal schemes from discovery"
      priority: 8
  initial-beliefs:
    - key: "penelope-awareness"
      value: "Penelope is naive and trusts too easily"
    - key: "peter-threat"
      value: "Peter Perfect is protective but predictable"
```

**Eidos API impact:** The `social:` section is parsed by wacky-manor, not by
eidos-api. Eidos descriptors support arbitrary extension data — the YAML is
loaded by `AgentRegistry` and the social section is extracted by a new
`SocialCognitionLoader` in wacky-manor. No eidos-api changes required.

### 2. Influence mode: Prompt-visible (D3)

Beliefs, drives, and norms are injected directly into the observation text
the LLM sees. The LLM reasons about them explicitly — no hidden pre-filtering.

The `ObservationBuilder` gains three new sections rendered by a
`SocialCognitionRenderer`:

```
## Your Drives
- SCHEMING (strong): You are compelled to hatch elaborate plans against Penelope
- SELF-PRESERVATION (moderate): You avoid direct confrontation, prefer subterfuge

## Your Beliefs
- You believe Penelope is naive and trusts too easily
- You believe Peter Perfect is protective but predictable
- [REVISED tick 12] You now believe the poison is in the Kitchen (was: Library)

## Your Norms
- NEVER help Penelope directly (priority: high)
- Maintain a veneer of charm in public (priority: medium)
- Protect personal schemes from discovery (priority: high)
```

Revised beliefs are marked with the tick they changed and the prior value,
so the LLM can reason about what changed and why.

### 3. Trust replacement

**Current:** `ManorTrustProvider` tracks global +/- scores per character.
STEAL → negative, GIVE → positive. No per-relationship dimension, no decay.

**Note:** blocks `trust` is vouch/intake-oriented (`IntakeClassifier`,
`VouchService`, `VouchEligibility`) — designed for onboarding trust, not
ongoing inter-character trust scoring. It doesn't fit wacky-manor's needs.

**Replacement:** New `ManorRelationshipTrust` in wacky-manor — richer than
the current `ManorTrustProvider` but Manor-specific, not blocks-based:

- **Per-relationship:** HC→Penelope trust is separate from HC→Peter trust
- **Event-weighted:** STEAL = -2, GIVE = +1, HELP = +0.5, BETRAY = -3
- **Decay:** trust drifts toward neutral (0.5) over time without events
- **Prompt-visible:** trust relationships appear in observations:
  ```
  ## Your Trust
  - Peter Perfect: HIGH (he's been helpful consistently)
  - Penelope Pitstop: LOW (she interfered with your last scheme)
  - Muttley: MODERATE (loyal but unreliable)
  ```

Trust updates happen in the same place as current `ManorTrustProvider` calls
in `ScenarioOrchestrator` — after action resolution, based on action type
and target. The new model is a drop-in replacement with richer state.

### 4. Disposition replacement

**Current:** `ManorDispositionRecorder` records behavioral signals.
`ManorPersonalityEvolution` checks for evolution periodically.

**Replacement:** blocks `memory` package:

- `SurpriseScorer` — flags unexpected events (HC being helpful = high surprise)
- `ArousalScorer` — high-stakes moments (STEAL, USE poison) get more cognitive weight
- `ConfidenceScorer` — tracks how confident a character is in their plans
- `MemoryHygieneOrchestrator` — automatic memory cleanup replacing manual decay config

The scoring feeds back into `AgentExperienceService.ingest()` — the importance
score calculation currently hard-coded in `importanceForAction()` would delegate
to the blocks scorers.

### 5. Belief store and revision

Uses blocks' `Belief`, `BeliefSet`, and `ConsistencyChecker` from
`blocks.agentic.belief` as the foundation. Blocks provides the data model
and consistency checking; wacky-manor adds revision triggers.

**Lifecycle:**
1. **Init:** Load `initial-beliefs` from Eidos descriptor at scenario start
2. **Per-tick:** Render current beliefs into observation sections
3. **Post-tick revision:** After action outcomes and observations from others,
   check for contradictions against current beliefs
4. **Revision:** When contradiction detected, update belief, mark as `[REVISED]`

**Belief revision triggers:**
- Character moves to a room and sees an item they believed was elsewhere
- Character observes another character doing something contradicting a belief
  about that character ("believed Muttley was loyal" → sees Muttley steal)
- Action failure that contradicts an assumption ("tried to use poison in Library"
  → "poison is not here" → revise location belief)

**Revision is rule-based, not LLM-driven.** The `BeliefRevisionService` checks
action results and world-state observations against stored beliefs using simple
pattern matching. This keeps revision deterministic and fast. The LLM sees the
revised beliefs in the next tick's observation and reasons about them.

### 6. Social drives

Drives use blocks' `DriveProfile` and `DriveConfig` from
`blocks.agentic.social.drive`. Blocks provides four core axes:
`AffiliationDrive`, `AutonomyDrive`, `CompetenceDrive`, `CuriosityDrive`
(self-determination theory). Each has intensity 0.0–1.0.

Wacky-manor character motivations (scheming, loyalty, dominance) don't map
cleanly to these four axes. Two options:

1. **Map to existing axes:** scheming ≈ high autonomy + high competence,
   loyalty ≈ high affiliation, curiosity maps directly.
2. **Use custom descriptive drives alongside blocks axes:** blocks provides
   `DriveConfig` for profile configuration — use the four axes for the
   underlying drive model, and add free-text `description` drives in Eidos
   for LLM-visible flavor.

**Approach: option 2.** Use blocks `DriveProfile` for the structured drive
model (4 axes with intensities). Add free-text `description` entries in
the Eidos `social.drives` section for character-specific flavor that the
LLM sees. `DrivePromptSection` from `blocks.agentic.social.prompt`
handles rendering the structured drives. The free-text descriptions are
rendered alongside.

Drives are static personality-level attributes — they don't change during
gameplay.

### 7. Normative reasoning

Norms are per-character behavioral rules loaded from Eidos descriptors.

Each norm has:
- `rule` — human-readable constraint
- `priority` — integer (higher = more important)

**Conflict resolution:** When the LLM's response would violate a norm, the
system doesn't enforce norms mechanically — the LLM sees them in the prompt
and is expected to self-regulate. Norms are advisory, not hard constraints.

However, blocks `normative` conflict resolution (`ConflictResolutionStrategy`)
is used to determine which norms to surface when multiple apply. If a
character has 10 norms but only 3 are relevant to the current situation,
the normative resolver filters and ranks them before rendering.

**Norm relevance:** Norms are filtered by context tags matching the current
situation:
- In a room with Penelope → norms tagged `penelope` surface
- Holding a weapon → norms tagged `violence` surface
- No matching tags → all norms rendered (default behavior)

---

## Tick Loop Changes

The autonomous tick loop in `ScenarioOrchestrator.runAutonomousTicks()` changes:

```
Per character per tick (changes marked ★):
  1. Drain observations (existing)
  2. Recall memories/reflections/relationships (existing)
  ★3. Load beliefs from BeliefStore
  ★4. Load drives from SocialCognitionLoader (cached per scenario)
  ★5. Filter norms by context via blocks normative resolver
  6. Build observation text (existing ObservationBuilder)
     ★ Append beliefs/drives/norms/trust sections via SocialCognitionRenderer
  7. Invoke LLM (existing)
  8. Process response: dialogue, actions (existing)
  ★9. Record trust via blocks trust (replaces ManorTrustProvider)
  ★10. Record disposition via blocks memory scoring (replaces ManorDispositionRecorder)
  ★11. Revise beliefs based on action outcomes
  12. Ingest experience (existing)
  13. Check personality evolution (existing — uses blocks scoring now)
```

---

## New Types

| Type | Package | Responsibility |
|------|---------|---------------|
| `SocialCognitionLoader` | `manor.agent` | Parses `social:` section from Eidos descriptors. Caches per scenario. |
| `BeliefStore` | `manor.agent` | Per-character in-memory belief map. CRUD + revision tracking. |
| `BeliefRevisionService` | `manor.agent` | Post-tick belief revision based on action outcomes and observations. |
| `SocialCognitionRenderer` | `manor.agent` | Renders beliefs, drives, norms, trust into observation sections. Uses blocks `SocialPromptAssembler` + `DrivePromptSection` from `blocks.agentic.social.prompt`. |
| `ManorRelationshipTrust` | `manor.agent` | Per-relationship trust with decay and event weighting. Manor-specific (blocks trust is vouch-oriented, doesn't fit). |
| `BlocksDispositionAdapter` | `manor.agent` | Adapts blocks memory scoring to the existing disposition call sites. |

## Removed Types

| Type | Replaced by |
|------|-------------|
| `ManorTrustProvider` | `ManorRelationshipTrust` |
| `ManorDispositionRecorder` | `BlocksDispositionAdapter` |
| `ManorPersonalityEvolution` | blocks `MemoryHygieneOrchestrator` via `BlocksDispositionAdapter` |

## Modified Types

| Type | Change |
|------|--------|
| `ScenarioOrchestrator` | Wire new services, replace trust/disposition instantiation |
| `ObservationBuilder` | Accept `SocialCognitionRenderer`, append new sections |
| `AgentExperienceService` | Delegate importance scoring to blocks scorers |

---

## Character Social Profiles

Five core characters get social configuration:

**Hooded Claw:**
- Drives: scheming (0.9), self-preservation (0.7), dominance (0.6)
- Norms: never help Penelope, maintain charm in public, protect schemes from discovery
- Initial beliefs: Penelope is naive, Peter is predictable, poison is in the Library

**Penelope Pitstop:**
- Drives: cooperation (0.8), curiosity (0.7), self-preservation (0.5)
- Norms: help anyone in need, avoid violence, trust until proven wrong
- Initial beliefs: everyone is fundamentally good, the manor has a mystery to solve

**Peter Perfect:**
- Drives: loyalty (0.9), cooperation (0.6), self-preservation (0.4)
- Norms: protect Penelope, confront threats directly, never steal
- Initial beliefs: Hooded Claw is suspicious, Penelope needs protection

**Muttley:**
- Drives: loyalty (0.7), self-preservation (0.8), curiosity (0.5)
- Norms: follow Dastardly's lead, avoid direct confrontation, hoard interesting items
- Initial beliefs: Dastardly has a plan, the other characters are unpredictable

**Dick Dastardly:**
- Drives: scheming (0.7), dominance (0.8), self-preservation (0.6)
- Norms: maintain authority over Muttley, outdo the Hooded Claw, never appear weak
- Initial beliefs: Hooded Claw is a rival schemer, Penelope is an obstacle

---

## Testing Strategy

**Unit tests (deterministic):**
- `BeliefStoreTest` — CRUD, revision tracking, initial belief loading
- `BeliefRevisionServiceTest` — revision triggers, pattern matching, edge cases
- `SocialCognitionLoaderTest` — YAML parsing, validation, caching
- `SocialCognitionRendererTest` — correct section formatting, revised belief markers
- `BlocksTrustAdapterTest` — per-relationship scoring, decay
- `BlocksDispositionAdapterTest` — surprise/arousal/confidence scoring delegation

**Integration tests (deterministic):**
- `ObservationBuilderIntegrationTest` — full observation with social sections
- `ScenarioOrchestratorSocialTest` — verify social cognition wired into tick loop

**LLM eval tests (non-deterministic, `@Tag("llm-eval")`):**
- `SocialCognitionEvalTest` — verify that social cognition inputs change character behavior:
  - HC with scheming drive acts more aggressively than without
  - Penelope with cooperation norms helps more than without
  - Belief revision after seeing moved item changes subsequent actions
  - Trust affects willingness to cooperate

---

## What's NOT in Scope

- **Goal/plan system changes** — keep existing ManorGoal*/ManorPlan* systems
- **Memory backend change** — Phase B (mindmap) is a separate issue
- **RAG, CBR, speech** — not relevant for this demo
- **PatternSpec YAML** — wacky-manor benefits from readable Java
- **Norm enforcement** — norms are advisory (prompt-visible), not hard constraints
- **Dynamic drives** — drives are static personality attributes for Phase A
- **LLM-driven belief revision** — revision is rule-based for determinism and speed

---

## References

- `ScenarioOrchestrator.java` — current tick loop architecture
- `ManorTrustProvider.java` — trust subsystem being replaced
- `ManorDispositionRecorder.java` + `ManorPersonalityEvolution.java` — disposition subsystem being replaced
- `ObservationBuilder.java` — observation rendering being extended
- `blocks/blocks/.../trust/` — blocks trust package
- `blocks/blocks/.../memory/` — blocks memory scoring (SurpriseScorer, ArousalScorer, etc.)
- `blocks/blocks/.../normative/` — blocks normative reasoning
- `blocks/blocks/.../agentic/social/` — blocks social cognition
- `blocks/blocks/.../agentic/belief/` — blocks belief revision
- `GE-20260816-6635e1` — ChannelObserver bridging pattern (may inform future channel-based belief propagation)
- `POC-SPEC.md` — phase structure and verdict gates
- D1–D4 in `decisions.md` — captured design decisions
