## D1: Implementation strategy — sequential experimentation

**Choice:** Try Approach 2 (LLM system prompt improvement) first, then Approach 3 (CognitiveDerivationEngine continuous weights). Phase 2 architecture deferred — its design will be informed by Phase 1 eval results and literature review, not upfront guesswork.
**Alternatives:**
- Approach 1 only (temporal RM pinning) — targeted shame fix but doesn't address 4→5 ceiling
- Pick one and commit — misses learning opportunity; empirical evidence should drive decisions
**Rationale:** Sequential experimentation lets each approach inform the next. Approach 2 is low-cost and reveals what LLM guidance can achieve, which calibrates how much mechanical precision Approach 3 needs. Findings documented in neocortex research docs.
**Trade-offs:** More total implementation time than picking one; risk of approach 2 being sufficient and approach 3 becoming unnecessary (acceptable — that's a good outcome)
**Sources:** Issue #830 analysis, 8 judge runs from #818 session
**Exploration:** quick
**Status:** captured

## D2: Pass disposition axes to LLM on predictor fall-through

**Choice:** Include personality disposition axes in the user message when the predictor falls through
**Alternatives:**
- System prompt only — generic shame guidance without personality context, less targeted
**Rationale:** The LLM needs the same personality information the predictor had to make personality-congruent strategy choices. The predictor falls through because no single strategy is mechanically predictable — but that doesn't mean personality is irrelevant to the choice.
**Trade-offs:** Slightly more LLM input tokens; the LLM may not use the axes effectively (eval will tell us)
**Sources:** EmotionRegulationGenerationPhase.java:155-232, RegulationStrategyPredictor.java:51-60
**Exploration:** quick
**Status:** captured

## D3: Shame guidance placement — conditional user message

**Choice:** Inject self-conscious emotion guidance as a conditional section in the user message (only when SHAME or REMORSE is dominant), not in the system prompt
**Alternatives:**
- System prompt (always visible) — simpler code but adds tokens to every regulation call
**Rationale:** Matches the existing conditional injection pattern used for fatigue, cognitive load, and strategy failure signals in `buildUserMessage()`. Keeps the system prompt generic; emotion-type-specific guidance only appears when relevant.
**Trade-offs:** Code must detect self-conscious emotions in the user message builder, adding a dependency on EmotionType classification
**Sources:** EmotionRegulationGenerationPhase.java:170-231 (conditional sections pattern), Webb et al. 2012 (strategy effectiveness varies by emotion type), Nathanson compass of shame (temporal avoidance→reappraisal sequence), Yang et al. 2025 (neural evidence for temporal suppression dynamics in avoidant attachment)
**Exploration:** deep-analysis
**Status:** captured

## D4: Phase 2 architecture — deferred to empirical evidence

**Choice:** Defer the CognitiveDerivationEngine architecture decision (replace boolean vs. augment) until Phase 1 eval results and literature review are available
**Alternatives:**
- Pre-decide now — requires guessing at what continuous weights should look like without knowing what the LLM can and can't handle
**Rationale:** Phase 1 results will reveal which personality×emotion combinations the LLM handles well with guidance vs. which need mechanical prediction. This directly informs whether continuous weights should replace or augment the boolean classification.
**Trade-offs:** Phase 2 requires its own design cycle; can't be planned in advance
**Sources:** User direction: "can we look at this through first principles and ensure it's a decision made in empirical testing and literature"
**Exploration:** quick
**Status:** captured
