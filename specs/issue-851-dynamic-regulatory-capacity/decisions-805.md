# Decisions — #805 Social Emotion Regulation

## D1: Scope — full Zaki & Williams taxonomy

**Choice:** Both intrinsic (using others to regulate own emotions) and extrinsic (regulating others' emotions)
**Alternatives:**
- Intrinsic only — simpler, natural first step, but misses the most dramatic social interactions (co-regulation, soothing)
- Extrinsic only — unusual starting point, lacks the self-regulation foundation
**Rationale:** The existing architecture (RegulationTracker for observation, RegulationContext for social context, appendGoals for additive goal semantics) already supports both directions. Implementing together avoids a second pass through the same code paths.
**Trade-offs:** Larger scope — more strategies to implement, more test scenarios, more prompt engineering
**Sources:** Zaki & Williams 2013, issue #805 description, RegulationTracker.java, RegulationContext.java
**Exploration:** quick
**Status:** captured

## D2: Taxonomy — extend existing EmotionRegulationStrategy enum

**Choice:** Add social strategies as new enum values to EmotionRegulationStrategy
**Alternatives:**
- Separate SocialRegulationStrategy enum — cleaner separation but doubles the pipeline (separate participant, goal bridge, LLM call)
- Composite enum + qualifier — theoretically elegant but confuses the LLM prompt with mixed semantics
**Rationale:** The #590 spec explicitly states the enum is "extensible." One decision layer, one result type, one goal bridge. The LLM sees all strategies in a single prompt and picks the best one for the situation.
**Trade-offs:** Enum grows from 5 to ~10 values. `isEnvironmentalStrategy()` needs updating or a companion `isSocialStrategy()`.
**Sources:** EmotionRegulationStrategy.java, #590 spec §Architecture, EmotionRegulationGenerationPhase.java
**Exploration:** quick
**Status:** captured

## D3: Target specification — add targetAgentId to EmotionRegulationResult

**Choice:** Add a nullable `targetAgentId` field to EmotionRegulationResult
**Alternatives:**
- Encode target in environmentalDirective text — fragile, requires parsing agent IDs from free text
- New SocialRegulationDirective record — most precise but adds a new type to the API
**Rationale:** Social strategies inherently target specific agents. A structured field is parseable by downstream consumers (goal bridge, prompt renderer). The LLM can name specific agents because RegulationContext.interactingAgents() provides the available targets.
**Trade-offs:** EmotionRegulationResult gains a 10th field — manageable but the record is getting large
**Sources:** EmotionRegulationResult.java, RegulationContext.java, EmotionRegulationGoalParticipant.java
**Exploration:** quick
**Depends on:** D2 (social strategies use the same result type)
**Status:** captured

## D4: Extrinsic trigger — appraisal-driven

**Choice:** Extrinsic regulation triggers through the existing appraisal pathway — the character appraises another's distress as emotionally relevant to themselves
**Alternatives:**
- Separate extrinsic trigger — new participant checking RegulationTracker directly, bypassing appraisal gate. More responsive but parallel pathway
- Combined gate — appraisal + RegulationTracker WORSENING. Prevents helping improving targets but adds complexity
**Rationale:** Uses the existing activation gate. The character's empathic response to observing distress IS an appraisal event — it naturally triggers regulation. RegulationTracker trajectory is provided as LLM context, not as a mechanical gate.
**Trade-offs:** A character won't try to regulate another's emotions unless their own appraisal fires — no "cold" helping behavior. This is arguably correct (Zaki & Williams: interpersonal regulation is emotionally motivated).
**Sources:** AppraisalTickParticipant.java, RegulationTracker.java, EmotionRegulationTickParticipant.java §tick()
**Exploration:** quick
**Status:** captured

## D5: Strategy selection — unified with social gate

**Choice:** One LLM call considers all strategies (intrapersonal + social). Social strategies only appear in the prompt when RegulationContext has interactingAgents.
**Alternatives:**
- Separate social decision layer — second LLM call after intrapersonal. Allows simultaneous strategies but doubles cost
- Social-first priority — try social before intrapersonal. Biases toward drama but loses the "best strategy for this situation" judgment
**Rationale:** The LLM can weigh social vs. intrapersonal strategies holistically. A character alone can't seek comfort. When others are present, the LLM considers whether seeking support or self-regulation is more appropriate given personality. No additional LLM calls.
**Trade-offs:** Only one strategy per tick — can't simultaneously suppress AND seek comfort. Acceptable: the single-strategy model is already validated.
**Sources:** EmotionRegulationGenerationPhase.java §SYSTEM_PROMPT, RegulationContext.java
**Depends on:** D2 (social strategies in same enum means same prompt)
**Status:** captured

## D6: Personality modulation — extend RegulationDefaults

**Choice:** Add `socialRegulationTendency` field to RegulationDefaults, derived from agreeableness + affiliation disposition in CognitiveDerivationEngine
**Alternatives:**
- Use raw Big Five directly — let LLM judge from personality scores. Less mechanical control
- New SocialDisposition record — most precise but requires new derivation logic and new CognitiveDefaults field
**Rationale:** Keeps the existing derivation pipeline. High socialRegulationTendency → prefers social strategies when available. Low → prefers intrapersonal. Consistent with how redirectTendency already modulates proactive regulation.
**Trade-offs:** Adds complexity to CognitiveDerivationEngine derivation. The "right" derivation formula needs empirical tuning.
**Sources:** RegulationDefaults.java, CognitiveDerivationEngine.java, #819 spec §Personality Modulation
**Depends on:** D5 (personality modulates strategy selection in the unified prompt)
**Status:** captured
