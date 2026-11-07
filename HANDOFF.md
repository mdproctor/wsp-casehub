# HANDOFF — casehub-examples (slot 196)

## Last Session

**Issue:** #830 — regulation predictor quality
**Phase 3: boundary case resolution. mixed-anger resolved to 5. suppressor-shame improved to SS(4). compounding-shame remains at 3-4 — genuine psychological bind.**

### What was built this session (8 commits)

**Architecture (4 changes, all literature-grounded):**
1. **CC viable alternative check** (`VIABLE_ALTERNATIVE=0.45`) — when CC is dominant for controllable emotions but the personality has viable suppression (≥0.45) or redirect (≥0.45), predictor falls through. Based on Ford & Troy 2019 regulatory flexibility: personalities with multiple viable strategies should not be pinned; the situation should determine choice. This is the symmetric counterpart to the existing RM viable reappraisal check.

2. **Avoidant shame → SITUATION_SELECTION** — Nathanson 1992 compass of shame identifies withdrawal as the avoidant shame response. Previous guidance said "Choose ATTENTIONAL_DEPLOYMENT" (redirect attention). Strategy exploration proved the judge prefers SITUATION_SELECTION (full withdrawal from triggering context, score 5) over AD (score 4). The avoidant personality doesn't just look away — they leave.

3. **Serial regulation fatigue** — Gross & John 2003: habitual strategies show diminishing returns after repeated use. Compounding shame guidance now signals "your default is failing — shift strategies."

4. **Compound-avoidant escalation** — When both avoidant shame (→SS) and serial fatigue (→shift) fire for compounding cases, explicit escalation guidance to SS. SM was removed after strategy exploration showed judge scores SM at 1 for this personality ("contradicts coping trajectory").

**Eval infrastructure (3 new tests):**
- `StrategyExplorationTest` — tests ALL 5 Gross strategies against the judge for each boundary case. Pure strategy-label scoring, no generation. This was the breakthrough: it reveals the judge's true preference independent of narrative quality.
- `PipelineObservationTest` — full pipeline visibility: predictor → LLM response (strategy, narrative, effectiveness, environmental directive) → judge score. Shows exactly where scores are won or lost at each stage.
- `GenerationPromptDiagnosticTest` — prints the exact user message sent to the generation LLM. No LLM calls, instant. Shows what the LLM is working with.

**Also:** Added `situationContext` parameter to generation phase so the LLM sees the same trigger information the eval judge has. Fixed pre-existing compilation error in `RegulationPredictorBackTest`.

### What works and why (hypotheses supported by evidence)

**mixed-anger CC=5:** The CC viable alternative check makes the predictor fall through (instead of pinning CC). Temporal staging fires with the original "Choose COGNITIVE_CHANGE" directive, producing a narrative that shows RM→CC progression. The LLM generates: RM impulse ("clamping down") → CC engagement ("reappraisal, perspective-taking on the betrayer"). The judge scores 5 because the cooperative personality's perspective-taking IS reappraisal, and the temporal progression makes it feel personality-congruent rather than forced. **Hypothesis:** the key ingredient is the temporal staging narrative, not the strategy label alone. When CC is pinned without temporal staging (Phase 2), the LLM generates flat reappraisal → judge objects. When CC comes through temporal staging (Phase 3), the narrative shows the personality's natural process → judge approves.

**suppressor-shame SS=4:** The judge explicitly prefers SITUATION_SELECTION over ATTENTIONAL_DEPLOYMENT (5 vs 4 in strategy exploration). SS produces an environmental directive (COMPETENCE axis: "seek low-stakes environment where competence is not under evaluation"). This is architecturally richer than AD — the regulation has behavioural consequences. **Hypothesis:** the remaining variance (4 vs 3 across runs) comes from the judge sometimes objecting that the cooperative orientation might allow CC. This is inherent in the personality profile (avoidant + cooperative = mixed signals).

**compounding-shame SS=3-4:** The LLM correctly identifies SS as the escalation from AD, and generates a narrative about restructuring the environment. But the judge sees 4 episodes of avoidance-family strategies as a maladaptive spiral. **Hypothesis:** no SINGLE Gross strategy scores 5 for this case because the personality's regulation capacity is genuinely exhausted. CC=1 (can't execute), SM=1 (contradicts avoidance), RM=3 (serial suppression failing), AD=4, SS=4. The model is correctly representing a clinical phenomenon: learned helplessness after repeated regulation failure.

### What doesn't work and why (hypotheses to test)

**Temporal staging softening failed.** Attempted to make the CC directive situation-dependent ("If betrayal, impulse response may be the appropriate endpoint"). This gave the LLM too much latitude — it stopped picking CC for cases where CC IS right (incoherent-anger 4→3). Reverted to original directive. **Hypothesis:** the LLM can't reliably distinguish "reappraisable" from "informationally valid" emotions from a short situation description. The temporal staging directive needs to be firm; situation-awareness should come from the narrative quality, not strategy selection.

**SM for compounding avoidant was wrong.** Strategy exploration showed judge scores SM at 1 ("contradicts personality's coping trajectory"). An avoidant personality with low autonomy cannot proactively restructure situations. **Hypothesis:** the architectural gap for compounding isn't missing strategies — it's that single-strategy regulation is insufficient. After 4 episodes, the personality needs either: (a) a multi-strategy sequence within a single regulation episode, or (b) explicit acknowledgment that regulation capacity is exhausted, shifting to an acceptance/tolerance mode.

**Judge variance is irreducible at boundaries.** The same strategy label + personality + emotion combination produces different scores across runs. This is inherent in LLM non-determinism. **Hypothesis:** the judge's score depends on which ASPECT of the personality it attends to in a given run. For mixed-anger CC: sometimes the judge focuses on "cooperative → perspective-taking → CC is natural" (→5), sometimes on "betrayal resists reframing" (→3). Both are psychologically valid readings.

## Next Action

**`work continue` on #830 — stabilise remaining boundary cases.**

### Approaches to take next

The strategy exploration + pipeline observation methodology proved valuable. Continue with this empirical approach:

#### Approach 1: Multi-strategy regulation for compounding-shame

The compounding case fails because no single strategy scores 5. A two-phase regulation model could address this: the character TRIES SS (their instinct), recognises it's failing (episode count signals this), then shifts to a qualitatively different mode — acceptance rather than regulation. This maps to "acceptance and commitment therapy" (ACT) approaches where regulation failure itself becomes the therapeutic signal. **Test:** modify the generation phase to output a two-phase narrative for compounding cases and see if the judge evaluates the SECOND phase more favourably.

#### Approach 2: Capacity exhaustion as a valid outcome

Instead of forcing a strategy, model exhaustion explicitly: the character's regulatory resources are depleted after 4 episodes, and the emotion breaks through. This is the "regulatory breakdown" concept from Baumeister's ego depletion model (though ego depletion's replication record is mixed). The generation phase could produce a narrative where regulation FAILS and the raw emotion is experienced — which is itself psychologically valid. **Test:** add a "capacity exhaustion" strategy type or modify the eval to accept breakthrough as an outcome.

#### Approach 3: Judge prompt evolution

The current judge evaluates STRATEGY SELECTION only (the label). It doesn't see the narrative. Adding the narrative to the judge prompt might reduce variance for cases like mixed-anger where the temporal progression IS the point. **Test:** create a variant judge prompt that includes the narrative and evaluates "strategy selection AND quality of execution." Compare scores with the current label-only judge.

#### Approach 4: Run strategy exploration at scale

The current exploration tested each strategy once per case. Running 5× per strategy would give a distribution rather than a point estimate. This would reveal which strategies are STABLY high-scoring vs which swing. Particularly useful for suppressor-shame (is SS really 5 or is it 4-5?) and compounding-shame (is SS really 4 or 3-4?). **Test:** modify StrategyExplorationTest to run N iterations and report mean ± variance.

#### Approach 5: Situation-type taxonomy

mixed-anger's variance comes from the judge's attention to betrayal vs personality. A situation-type taxonomy (frustration, betrayal, moral violation, relational wound) could allow the predictor to make situation-specific predictions without falling through. This is Troy et al. 2013's "person-by-situation" approach. **Test:** add a small number of situation types and test whether the predictor can improve scores by incorporating situation into strategy selection.

The next session should feel free to expand on these, combine them, or develop entirely new approaches. The pipeline observation tooling makes it possible to test ideas quickly — the prompt diagnostic test runs instantly, the strategy exploration runs in minutes, and the full pipeline observation takes about a minute per case.

### Files to read first
- `docs/research/2026-11-07-regulation-continuous-weights-literature-review.md` — full literature review + Phase 2 calibration + Phase 3 strategy exploration
- `docs/eval/strategy-exploration-phase3.json` — judge scores for all 5 strategies × 3 boundary cases
- `docs/eval/regulation-e2e-phase3-run3.json` — latest E2E eval with narratives
- `cognition/.../regulation/RegulationStrategyPredictor.java` — predictor with CC viable alternative check
- `cognition/.../regulation/EmotionRegulationGenerationPhase.java` — generation phase with SS guidance + situation context
- `cognition/.../regulation/GenerationPromptDiagnosticTest.java` — prints exact LLM prompts (run first!)
- `caps-llm-classifier/.../eval/StrategyExplorationTest.java` — tests all strategies against judge
- `caps-llm-classifier/.../eval/PipelineObservationTest.java` — full pipeline visibility

## Branches

**Neocortex:** `issue-830-regulation-predictor-quality` — 24 commits (Phase 1 + Phase 2 + calibration + Phase 3)
**Examples:** `issue-830-regulation-predictor-quality` — no code changes
**Workspace:** `issue-830-regulation-predictor-quality` — specs, plans, decisions, blog entries

## Queue Status

Issue #830 active. 11 remaining issues after #830 (including #844 feedback loop).
