# HANDOFF — casehub-examples (slot 196)

## Last Session

**Issue:** #830 — fix: regulation predictor quality — address shame fall-through and 4→5 ceiling
**Phase 1 complete. Phase 2 pending.**

Implemented personality-aware LLM guidance for predictor fall-through cases. Two conditional sections added to `EmotionRegulationGenerationPhase.buildUserMessage()`:

1. **Disposition axes injection** — when predictor returns `Optional.empty()`, the LLM receives the character's five disposition axes (conflict-mode, social-orient, risk-appetite, autonomy, rule-following) with explicit "predictor did not pin a strategy" framing.

2. **Self-conscious emotion guidance** — when dominant emotion is SHAME/REMORSE and no strategy is pinned, the LLM receives literature-grounded temporal containment guidance (Webb 2012, Nathanson 1992, Yang 2025).

Wired through `EmotionRegulationTickParticipant.tick()` → new 9-param `generate()` overload → new 9-param `invokeLlm()` → new 10-param `buildUserMessage()`.

### Eval Results
- Predictor eval: mean 5.0 (was 4.6), no regressions, all 5 scored cases at 5
- suppressor-shame still falls through (predictor logic unchanged — improvement is in LLM path)
- The 4→5 improvement is likely LLM judge variance, not caused by Phase 1 changes
- **Eval gap:** current eval judges predictor selection only, not LLM response quality on fall-through

### Blog
- New entry: `blog/2026-11-06-mdp01-when-the-predictor-lets-go.md` — third in regulation series
- Continues from "When Personality Isn't Enough" (2026-11-05)

## Branches

**Neocortex:** `issue-830-regulation-predictor-quality` — 5 commits:
1. `feat(#830): add disposition axes to regulation user message on predictor fall-through`
2. `feat(#830): add self-conscious emotion guidance for predictor fall-through`
3. `feat(#830): wire disposition axes through tick participant to generation phase`
4. `eval(#830): Phase 1 regulation quality findings — personality-aware fall-through guidance`
5. `docs(#830): expand Phase 1 findings with full nuance and literature detail`

**Examples:** `issue-830-regulation-predictor-quality` — no code changes (all changes in neocortex)

**Workspace:** `issue-830-regulation-predictor-quality` — specs, plans, decisions, blog

## Phase 2: What Needs to Happen Next

### Step 1: Build end-to-end regulation eval (PREREQUISITE)

The current eval (`RegulationPredictorBackTest`) only judges the predictor's strategy selection. Phase 1 improved the LLM path — what happens when the predictor falls through. That improvement is invisible to the current eval.

Build an end-to-end eval that:
- Runs the full pipeline (predictor → LLM) for each personality × emotion combination
- Judges the LLM's chosen strategy (not the predictor's)
- Includes fall-through cases (suppressor+shame, incoherent+anger, mixed+anger)
- Measures whether temporal containment appears in suppressor+shame narratives

**Why prerequisite:** Without measuring Phase 1's actual impact on the LLM path, Phase 2's architecture decision (replace boolean classification vs. augment with continuous weights) would be based on guesswork.

### Step 2: Brainstorm Phase 2 architecture

Based on end-to-end eval results, design `deriveRegulationDefaults()` for CognitiveDerivationEngine. Key decisions:
- Replace boolean `isSuppressor()`/`isReappraiser()` entirely, or augment with continuous weights as a gradient?
- What weight tables? (Need literature grounding — Hughes 2020, Petter 2025 provide the starting point)
- What thresholds turn continuous weights into strategy predictions?

**User direction:** "can we look at this through first principles and ensure it's a decision made in empirical testing and literature" — do not pre-decide architecture without evidence.

**Key insight for Phase 2 (NOT implemented — design observations only):** Shame has three dimensions the current predictor doesn't account for: (1) memory-triggered compounding (previous failures amplify current intensity via salience-scored retrieval), (2) relationship-modulated intensity (failing your child vs. a nephew — relational closeness changes what the self-evaluation is about), (3) personality-gated depth (Cluster-B grandiosity+callousness can produce surface-level shame that doesn't persist). The building blocks exist in the platform (memory system, social cognition model, PersonalityFacets/CDE expression styles) but the regulation pipeline doesn't consume them yet. See research doc observation #5 for full detail.

### Step 3: Implement Phase 2

Follows the standard flow: brainstorm → writing-plans → executing-plans.

## Key Files to Read

### Research (comprehensive context)
- `neocortex/docs/research/2026-11-06-regulation-predictor-quality-phase1-findings.md` — **READ THIS FIRST.** Full problem statement, literature grounding (5 papers), eval results with tables, eval gap analysis, and 5 specific observations for Phase 2.
- `neocortex/docs/research/2026-11-05-post-era6-regulation-and-identity.md` — regulation system overview, architecture, cascade dynamics

### Design artifacts
- `wsp-casehub/specs/issue-830-regulation-predictor-quality/2026-11-06-regulation-predictor-quality-design.md` — Phase 1 design spec
- `wsp-casehub/specs/issue-830-regulation-predictor-quality/decisions.md` — 4 decisions (D1-D4), including D4 which explicitly defers Phase 2 architecture
- `wsp-casehub/plans/2026-11-06-regulation-predictor-quality.md` — Phase 1 implementation plan (4/4 tasks complete)

### Source code (what was changed)
- `neocortex/cognition/src/main/java/.../regulation/EmotionRegulationGenerationPhase.java` — new 10-param `buildUserMessage()` with axes + self-conscious guidance sections
- `neocortex/cognition/src/main/java/.../regulation/EmotionRegulationTickParticipant.java` — wiring: passes axes + dominant emotion type on fall-through
- `neocortex/cognition/src/main/java/.../regulation/RegulationStrategyPredictor.java` — UNCHANGED (uses `emotionCategory()` from generation phase, but logic untouched)

### Eval
- `neocortex/caps-llm-classifier/target/regulation-predictor-judge-results.json` — latest judge results (mean 5.0)
- `neocortex/caps-llm-classifier/src/test/java/.../eval/RegulationPredictorBackTest.java` — the predictor-only eval (needs end-to-end companion)

### Blog
- `blog/2026-11-06-mdp01-when-the-predictor-lets-go.md` — public-facing explanation of this work
- `blog/2026-11-05-mdp01-when-personality-isnt-enough.md` — predecessor (predictor coherence, shame)
- `blog/2026-11-04-mdp01-the-gap-between-feeling-and-acting.md` — first in series (regulation architecture)

### Literature (for Phase 2)
- Webb et al. 2012 — strategy effectiveness meta-analysis (d+ values)
- Nathanson 1992 — compass of shame, temporal containment
- Yang et al. 2025 — ERP evidence for temporal suppression dynamics
- Hughes et al. 2020 — personality × regulation stages (Big Five → EPM)
- Petter et al. 2025 — personality-strategy correlations (predictor foundation)

## Queue Status

Issue #830 is active in `.plan`. All Phase 1 tasks checked off. The queue (`.queue`) has 10 remaining issues after #830.

## Protocol Reminder

`cite-literature-for-model-constructs` — any weight tables in the CDE `deriveRegulationDefaults()` pathway must cite published literature. No arbitrary numbers.
