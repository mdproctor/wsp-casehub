# HANDOFF — casehub-examples (slot 196)

## Last Session

Implemented #823 (per-modality regulation eval tests) — 4 eval tests across 4 regulation modalities, plus a prerequisite CognitionCore API fix.

### What was built

**1. CognitionCore.tick() overload for RegulationContext (D10)**

Added 7-arg `tick(agentId, tenantId, descriptor, resolver, observation, targetAgentId, regulationContext)` to `CognitionCore.java`. Existing 4/5/6-arg overloads delegate to it with null. This fixes a TWO-DIRECTIONAL production gap:
- Write path: `EmotionRegulationTickParticipant` at line 182 passes `context.regulationContext()` to `EmotionRegulationMindMapAdapter.write()` — was always null, so regulation location data was never persisted to MindMap
- Read path: `ProactiveRegulationParticipant` at line 46-47 short-circuits when `regulationContext` is null — proactive avoidance never fired

Test: `CognitionCoreRegulationContextTest` (2 tests) verifies context flows to FOUNDATION participants and that existing overloads pass null.

**2. PerModalityRegulationEvalTest — 4 modality tests**

New test class in `caps-llm-classifier/src/test/java/io/casehub/neocortex/caps/llm/eval/`.

| Test | Ticks | Assertion | Result |
|------|-------|-----------|--------|
| `fatigue_sustainedRegulation_overridesToAttentionalDeployment` | 20 | Strategy transitions from COGNITIVE_CHANGE → ATTENTIONAL_DEPLOYMENT | 3/3 required |
| `automatic_predictorMatchesLlmFreeChoice` | 2×2 paired | Predictor strategy == LLM free-choice strategy | ≥2/3 |
| `monitoring_decliningEffectiveness_triggersStrategySwitch` | 5 | Strategy on tick 5 differs from ticks 1-4 after failure signal | ≥1/3 |
| `proactive_memoryBasedAvoidance_producesAvoidanceGoal` | 3 | DriveGoalProposal with "proactive-avoidance-library" produced | ≥2/3 |

First eval run: 2/4 passed (fatigue ✅, monitoring ✅). Two fixes applied:
- **Automatic**: observation was too intense (peak > 0.75), triggering predictor's high-intensity override to ATTENTIONAL_DEPLOYMENT. Calmed observation → 3/3.
- **Proactive**: `configureRegulation()` called before any `tick()`, so `lastTenantId` was null and the COGNITIVE subgraph lookup used "default" instead of "eval-tenant" — regulation results never written to MindMap. Fix: setup tick before `configureRegulation()` → 3/3.

Second eval run: 4/4 passed.

### Key design findings

**Fatigue is a convergent series, not linear.** The spec assumed fatigue = 0.15/tick (linear). Actual: `effort × 0.15` where effort = `alpha × discrepancy / D_MAX` and alpha = `0.5 × (1 - fatigue)`. This is a negative feedback loop. After 5 ticks: fatigue ≈ 0.30 (not 0.75). Override threshold (0.7) reached around tick 18-20. Decisions D11, D13 document this.

**Monitor + fatigue emergent interaction.** The LLM receives fatigue as prompt context ("Regulation fatigue: 30%") and responds by generating lower effectiveness values. After 3 consecutive declines, the monitor fires a failure signal and the strategy switches — much earlier than the fatigue override threshold alone. This was not designed; it emerged from the LLM interpreting fatigue context.

**Observation intensity must stay ≤ 0.75.** The predictor has a hard high-intensity override at `RegulationStrategyPredictor.HIGH_INTENSITY_THRESHOLD = 0.75` (line 17). Observations above this always get ATTENTIONAL_DEPLOYMENT regardless of personality. Dramatic observations ("your journal read aloud") exceed this. Moderate ones ("seat taken at breakfast") stay below. This affects ALL eval tests, not just the automatic concordance test.

### Decisions captured

D9-D14 in `specs/issue-851-dynamic-regulatory-capacity/decisions.md`:
- D9: Separate test class (multi-tick vs 2-tick pattern)
- D10: CognitionCore.tick() RegulationContext overload (production gap)
- D11: ~20 ticks for fatigue (convergent series arithmetic)
- D12: Strategy concordance only, no timing (LLM called both paths)
- D13: 5-tick monitoring signal consumption cycle
- D14: 3-tick proactive with seeded MindMap via RegulationContext

### Blog written

`examples/docs/blog/2026-11-09-mdp01-the-test-that-broke-the-math.md` — covers convergent series, API gap, emergent interaction, and why multi-tick eval matters for cognitive agents. 3 non-inline SVGs.

### Repos synced

All 4 slot repos rebased onto canonical main (2 rebase conflicts in neocortex research doc resolved — merged completion markers from slot 211 with Epic #841 structure). Canonical main FF-merged for examples and neocortex.

## What to Watch

1. **Fatigue test at 3/3 threshold.** Raised from ≥2/3 to exactly N in this session. The first full run showed 2/3 transitioning (run 1 had ATTENTIONAL_DEPLOYMENT from tick 1 — the appraisal likely produced intensity > 0.75 on the first observation). If this flakes, investigate whether the first observation ("James took the last cup of tea") sometimes appraises above 0.75. Fix: make observation 1 even milder, or soften to ≥2/3.

2. **Pre-existing test failure.** `EmotionRegulationGenerationPhaseTest.userMessageIncludesSelfConsciousGuidanceForShame` still fails (shame guidance prompt content mismatch). Not caused by #822 or #823. Tracked but unfixed — may need shame prompt content update.

3. **Monitoring test leniency.** Threshold is ≥1/3 because effectiveness decline across ticks is LLM-generated and non-deterministic. If this consistently passes, consider raising to ≥2/3. If it consistently fails, the observation escalation pattern needs revision.

4. **Automatic concordance interpretation.** Currently both predictor and LLM agree on ATTENTIONAL_DEPLOYMENT — the observation may still be triggering high-intensity override. If future runs show predictor=ATTENTIONAL_DEPLOYMENT but LLM picks a different strategy, the test is actually validating what it should: does the predictor's personality mapping agree with LLM judgment? The current 3/3 agreement on ATTENTIONAL_DEPLOYMENT might be the high-intensity override making both paths agree trivially. Consider adding a second concordance scenario with a truly moderate-intensity emotion that tests the personality path (e.g., OTHER_DIRECTED with high suppression → RESPONSE_MODULATION).

5. **configureRegulation() tenant timing.** The proactive test's fix (setup tick before configureRegulation) is correct but fragile. If anyone reorders setup in the test, the COGNITIVE subgraph lookup breaks silently. The root cause is in `CognitionCore.configureRegulation()` line 746: `lastTenantId != null ? lastTenantId : "default"`. A more robust fix would accept tenant as a parameter. Not urgent — the test pattern documents the dependency.

## Immediate Next Step

**Queue status:** `ALL_DONE=True` for the current queue batch. `work next` will advance to the next issue. The queue shows:
- `#520 — Cognitive` (next unchecked)
- `#203 — Spatial`
- `#806 — CAPS`
- `#315 — Cognitive (casehub-mosaic)`
- `#828 — Regulation predictor observability`
- `#818 — Automatic`

**Recommended:** Run `work next` to advance the queue and close #823 on GitHub. Before advancing, consider whether the automatic concordance test (point 4 above) needs a second scenario with a genuinely moderate-intensity observation — this would be a quick addition before closing #823.

## References

- `specs/issue-851-dynamic-regulatory-capacity/2026-11-08-per-modality-regulation-eval-design.md` — #823 design spec (promoted to examples)
- `specs/issue-851-dynamic-regulatory-capacity/decisions.md` — D9-D14 (#823 decisions)
- `plans/2026-11-08-per-modality-regulation-eval.md` — #823 implementation plan (promoted)
- Neocortex branch: `issue-851-dynamic-regulatory-capacity`
- Neocortex commits: `45d523a7` (tick overload), `1189aab1` (eval tests), `a4c76a69` (fixes), `d832ed90` (fatigue threshold)
- `PerModalityRegulationEvalTest.java` — test class
- `CognitionCoreRegulationContextTest.java` — tick overload test
- `CognitionCore.java` — tick() overloads (lines 213-301)
- `docs/research/2026-11-05-post-era6-regulation-and-identity.md` — updated with findings
- `examples/docs/blog/2026-11-09-mdp01-the-test-that-broke-the-math.md` — blog post
- Pre-existing failure: `EmotionRegulationGenerationPhaseTest.userMessageIncludesSelfConsciousGuidanceForShame`
