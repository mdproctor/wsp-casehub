# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-18a:** Closed #1098 — replaced keyword intent classifier with LLM-resolved IntentValence. Filed #1112 (eval test for valence classification quality). Blog: "The Second Time Keywords Lied". Updated era5, era7 research docs and mental-model design spec.

**All branches landed and published.** Slot 196 marked ready for archive.

## What Was Landed This Session

### #1098 — Replace keyword intent classifier with BDI-dimension semantic classification (neocortex)

**3 commits landed:**

1. **feat:** Added `IntentValence` enum (HOSTILE, COOPERATIVE, NEUTRAL) to `AttributedState`. `classifyIntentBias` now reads the valence field directly instead of substring matching. Removed `isHostile`, `isCooperative`, `containsAny`. LLM extraction prompt updated to include valence in JSON schema. `formatBdi` includes existing entries' valence in re-inference prompts. 6-arg backward-compatible constructor defaults to NEUTRAL. Serialization handles missing valence gracefully.

2. **fix:** `decayMap` was using the 6-arg constructor, silently resetting LLM-classified valence to NEUTRAL during confidence decay. Fixed to preserve `s.valence()`.

3. **docs:** Updated era5-relational-cognition-research, era7-personality-trait-integration-research, and mental-model-design spec to reflect the IntentValence approach.

### Blog promoted to examples
- `docs/blog/2026-11-18-mdp01-the-second-time-keywords-lied.md` — the semantic resolution boundary applied to BDI intent classification

## Issues Filed This Session

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1112 | Eval test for LLM valence classification quality | S | Med | Verify the LLM produces correct hostile/cooperative/neutral labels across BDI scenarios |

## Queue — 2 items remaining

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #1089 | Unbounded cache growth | XS | Low | Deferred — no impact with fixed 17-character cast |
| #1097 | Personality-modulated CPTs for TheoryOfMindEngine | M | Med | Design-heavy — personality-driven Bayesian priors for ToM |

#1089 is deferred until dynamic agent creation becomes a requirement. #1097 is M/Med design work — an anxious agent should weight threat evidence more heavily, a callous agent should have different priors about others' emotional states. Needs literature review on personality-modulated Bayesian networks. Fresh session recommended.

## Coping Keyword Classifier — Flagged for Future

The blog's closing paragraph flags the coping micro-loop in emotion regulation as a third candidate for the same keyword → semantic resolution treatment. `RelationalSchemaEvolutionPhase` is now clean, but `CopingCheck` or similar may still use keyword detection for SEC re-evaluation triggers. Worth auditing when the coping layer is next touched.

## Slot Status

**Slot 196 marked ready for archive.** All canonical repos published. Cannot be archived from within the slot.

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s slot-settings.xml
```
