# HANDOFF — casehub-examples (slot 196)

## Last Session

Fixed the appraisal-in-tick failure. Root cause: `LlmAppraisalStrategy.parseResponse()` passed raw LLM output (including markdown ` ```json ` code fences from the Claude CLI) directly to Jackson, which threw `JsonProcessingException`. The exception was silently caught → returned `AppraisalResult.empty()`. Fix: extract JSON between `{` and `}` before parsing, consistent with `EmotionRegulationGenerationPhase.parseResponse()`.

Confirmed with `SessionVsInvokeReproducerTest` (3 tests): `invoke()`, pipeline `openSession()`, and bare `openSession()` all produce valid text — the session layer was never the problem.

After the fix, full end-to-end tick pipeline works: appraisal produces emotions (ANGER/SHAME/REPROACH), regulation activates with strategy selection (COGNITIVE_CHANGE, RESPONSE_MODULATION). Three per-scenario LLM eval tests confirm: betrayal (3/3 → RESPONSE_MODULATION), mortal danger (3/3 → COGNITIVE_CHANGE), public humiliation (3/3 on retry → COGNITIVE_CHANGE).

Also added `RegulationPipelineThroughTickTest` (4 mechanical tests, no LLM) proving the tick wiring with stubbed strategies: betrayal→anger, mortal danger→fear, public humiliation→reproach, low-intensity no-activation. All 576 cognition tests pass.

## Immediate Next Step

**Task 10 from the plan: A/B evaluation baseline run.** Run wacky-manor with regulation enabled/disabled using GENERIC profile, 300+ events, classify emotions, compare scores.

Deferred from previous sessions:
- R3-03: Cognitive load has no measurement signal — excluded from system prompt until a proxy exists
- File social regulation as separate issue referenced from #590

## References

- `wsp-casehub/specs/issue-590-emotion-regulation/2026-11-04-emotion-regulation-design.md` — design spec
- `wsp-casehub/plans/2026-11-04-emotion-regulation.md` — implementation plan (9/10 tasks complete)
- Neocortex branch: `issue-590-emotion-regulation` (18 commits — includes JSON fix + tick eval)
- Examples branch: `issue-590-emotion-regulation` (4 commits — config fix + wiring test)
- Key finding: EmotionMapper ANGER uses 3-factor product (compresses intensity); SHAME/REPROACH use 2-factor (better activation path for betrayal/humiliation scenarios)
