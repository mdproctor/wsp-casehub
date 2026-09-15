# SocialComparison Integration — Design Spec

**Issue:** casehubio/examples#60
**Epic:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Date:** 2026-09-15
**Status:** Draft

---

## Problem

Characters have no awareness of how their perceptions of nearby characters differ from those characters' self-perceptions. A schemer doesn't know that Peter sees himself as heroic while the schemer sees him as a predictable fool. This asymmetry is exactly the kind of social intelligence that drives interesting character behavior.

The platform has the infrastructure — `CognitiveProfile.compare()` returns per-agent perspectives on shared entities, and `SocialComparison.compare()` computes PAD distances, dimension differences, and trajectory alignment. Neither is wired into the observation pipeline.

---

## Design

### Integration Point

A new method `renderSocialAwareness()` in `CharacterCognition` that:

1. **Gates on drives** — only runs when the character has `scheming` or `suspicion` drives with intensity > 0.5 (from SocialConfig). Characters without these drives don't invest in theory-of-mind.

2. **Queries compare()** — for each nearby character X, calls `CognitiveProfile.compare(queryForX, Set.of(selfPrincipal, xPrincipal))`. Returns how the acting character sees X vs how X sees themselves.

3. **Computes divergence** — feeds the compare() result to `SocialComparison.compare()`. Gets `PerspectivalComparison` with `PadDistanceMatrix`.

4. **Filters by threshold** — only renders perception lines for relationships where PAD distance > 0.4 (significant divergence).

5. **Translates to perceptions** — maps the dominant PAD dimension difference to a character-perspective statement (see §Perception Translation below).

6. **Renders section** — produces an `ObservationSection.items("Social Awareness", null, lines)` with 1-2 lines per divergent relationship.

### Where It Lives in the Tick Loop

```
CharacterCognition.renderCognitiveSections()
  ├── "Your Drives" section
  ├── "Your Principles" section
  ├── "Your Beliefs" section
  ├── "Social Rules" section (norm scoring from #57)
  ├── "Cognitive State" section (CognitionCore)
  └── "Social Awareness" section ← NEW (this issue)
```

The Social Awareness section appears last — it's the highest-level cognitive output, synthesizing perspective comparisons across nearby entities.

### Perception Translation

For each nearby character with PAD distance > 0.4, find the PAD dimension with the largest absolute difference and render a perception statement:

| Dominant dimension | My value vs theirs | Template |
|-------------------|--------------------|----------|
| PLEASURE | mine < theirs | "{name} seems to view this more positively than you do" |
| PLEASURE | mine > theirs | "You feel more positively about {name} than they feel about themselves" |
| AROUSAL | mine > theirs | "You're more alert around {name} than they seem to be" |
| AROUSAL | mine < theirs | "{name} seems more on edge than you'd expect" |
| DOMINANCE | mine > theirs | "You feel more in control around {name} than they do" |
| DOMINANCE | mine < theirs | "{name} seems more confident in this interaction than you are" |

If trajectory data is available and shows DIVERGENT trend agreement, append: "(and this gap is widening)"

If trajectory shows ALIGNED, append: "(though you're converging)"

### Graceful Degradation

- **CognitiveProfile not available** (CDI Instance unresolvable) → no section rendered, no error
- **MindMapStore has no data** (early game, no interactions yet) → compare() returns empty map → no section
- **No PAD values on nodes** (nodes seeded without affect) → SocialComparison marks agents as `unassessed` → no section
- **No drives gate the character** → method returns empty list immediately
- **All PAD distances below threshold** → no section

The section appears only when there's meaningful data to show. This is the expected state for early game — it emerges naturally as interactions accumulate.

---

## Modified Types

### CharacterCognition

Add `renderSocialAwareness()` method. Called from `renderCognitiveSections()` after the existing sections.

**Dependencies for the method:**
- `cognitiveProfile` — already a field on CharacterCognition (from #54)
- `socialConfig` — already a field, checked for scheming/suspicion drives
- `tenantId` — already a field
- `agentId` → `PrincipalId.of(agentId)` — self principal
- Nearby agent IDs → `PrincipalId.of(nearbyId)` — nearby principals

The method needs to construct `CognitiveProfileQuery` per nearby character. The query requires:
- `entityName` — the nearby character's agent ID (used to resolve their mindmap node)
- `tenantId` — from CharacterCognition
- `asSeenBy` — the perspective principal (self or the nearby character)

### ManorContextStrategy

No changes. The drive gate is checked inside `renderSocialAwareness()` directly against `socialConfig.drives()`.

### CognitiveBudget

No changes. Social awareness doesn't use the budget — it has its own drive-based gate.

---

## Testing

### Unit Tests (no CDI, no mindmap)

| Test | Asserts |
|------|---------|
| noSectionWithoutSchemingOrSuspicionDrives | Character without scheming/suspicion drives → empty list |
| noSectionWhenCognitiveProfileNull | CognitiveProfile is null (CDI unavailable) → empty list |
| noSectionWhenNoNearbyCharacters | No nearby characters → empty list |
| perceptionTranslationPleasureLow | Pleasure difference → correct template selected |
| perceptionTranslationArousalHigh | Arousal difference → correct template selected |
| perceptionTranslationDominance | Dominance difference → correct template selected |
| belowThresholdFiltered | PAD distance 0.3 → no perception line |
| trajectoryDivergentAppended | Divergent trajectory → "(and this gap is widening)" appended |
| trajectoryAlignedAppended | Aligned trajectory → "(though you're converging)" appended |

These tests will use mock/stub CognitiveProfile and SocialComparison results. Since `renderSocialAwareness` calls these via the existing fields, tests construct CharacterCognition with a test-double CognitiveProfile.

### Integration Tests

| Test | Asserts |
|------|---------|
| socialAwarenessRenderedForSchemingCharacter | Hooded Claw (scheming 0.9) with populated mindmap → Social Awareness section appears |
| socialAwarenessAbsentForTrustingCharacter | Penelope (no scheming/suspicion) → no Social Awareness section |

---

## What's NOT in Scope

- Compare on self (how do nearby characters see ME) — can be added as a second pass
- Full social comparison UI — this is a minimal observation section, not a dashboard
- LLM-based perception description — template-based translation is sufficient; embeddings/LLM can be swapped later
- Bidirectional comparison — only outward perception (how I see X vs how X sees themselves)

---

## Design Decisions

- D1: Budget gate — drive-based (scheming/suspicion > 0.5)
- D2: Section content — perception-level translation (GE-20260915-2b5e80)
- D3: Compare target — each nearby character (how I see X vs X's self-image)

Full decision records: `specs/issue-60-social-comparison/decisions.md`

---

## References

- CognitiveProfile.compare() — neocortex cognitive-index, returns Map<PrincipalId, EntityKnowledge>
- SocialComparison.compare() — neocortex cognitive-index, returns PerspectivalComparison
- PerspectivalComparison — entityId, entityName, perspectives, distances, dimensionDifferences, trajectoryAlignment
- PadDistanceMatrix — distance(PrincipalId, PrincipalId), maxDistance(), meanDistance()
- TrajectoryAlignment — cosines, agreements (ALIGNED, DIVERGENT, MIXED, INSUFFICIENT)
- CharacterCognition.java:94-141 — renderCognitiveSections
- social-config.yaml — drive definitions (scheming, suspicion)
- #54 design spec §Per-Tick Query Flow step 3 — compare() budget gate description
- GE-20260912-ff141b — neocortex cognitive stack documentation
- GE-20260914-e3cb03 — profile data over prose directives (absolute state)
- GE-20260915-2b5e80 — comparative data perception-level translation (relational state)
