# HANDOFF — casehub-examples

## Last Session

Implemented appraisal-based drive filtering, ran three scenario
comparisons, and discovered that the reactive-drive problem is
fundamentally an eval methodology problem, not a rendering problem.

### What was done

**Appraisal filtering implemented (examples #122):**
- `CharacterCognition.appraisalFilterDrives()` — classify situation →
  read DriveProfileInferred → compute overlap → filter drives
- Reuses pattern from `selectSituationPrimedMemories`
- 5 graceful degradation paths (no classifier, no store, no text, empty
  activations, no profile)
- 15 unit tests including bidirectional falsification suite
- 3 LLM eval micro tests: door-stuck 5/5, dinner differentiation 5/5,
  competence reverse 5/5

**Neocortex revert + fix:**
- Reverted `cf3b0fa3` (ambient+acute prompt) — superseded by filtering
- Made `LlmSituationClassifier` constructor public for direct construction

**LLM classifier wired (neocortex #587):**
- `PlaybookOrchestrator.buildSituationClassifier()` constructs
  `LlmSituationClassifier` directly (not CDI — avoids ambiguity with
  rule-based producer used by consolidation pipeline)
- `LlmSituationClassifier` excluded from CDI discovery in
  application.properties
- Pipeline uses rule-based (fast, adequate for pre-classified formation
  memories). Render time uses LLM for semantic understanding.
- Concurrency increased: max-concurrent-sessions 6→12, gate max 5→10

**Filter observability:**
- `appraisalFilterDrives` logs: agent, situation activations, rendered
  drives, filtered drives per event
- Enables correlation between filter decisions and eval scores

**Three scenario runs:**
1. Rule-based classifier: PP gallantry 3.00 (quality 3.92, trigger 54%)
2. LLM classifier: PP gallantry 2.27 (quality 3.50, trigger 27%)
3. Both worse than B0 (4.50) because B0's vague descriptions inflated
   scores via drive contamination across irrelevant events

**Design principles established (Phase 8, #43-52):**
- #46: Appraisal filtering > ambient coloring
- #49: Score dominance in action, not exclusivity in thought
- #50: Eval must separate quality from trigger frequency
- #51: Filter decisions must be observable
- #52: Rule-based classification inadequate — LLM needed for context

**Issue created:**
- neocortex #587: Graduated SituationClassifier — NLP → LLM fallback →
  CBR confidence accumulation

### Key Finding — The Eval Methodology Problem

The reactive-drive problem was misdiagnosed. The real issue:

1. The eval scores EVERY drive in EVERY event and averages
2. Reactive drives correctly trigger in 20-30% of events
3. The other 70-80% score 1 (drive absent), dragging the mean to ~2
4. B0's "high" score (4.50) was inflated — vague descriptions made
   gallantry bleed into investigation events where it shouldn't express
5. The LLM classifier produces MORE correct behavior (gallantry only in
   genuine courtesy moments) but the mean DROPS because the eval
   penalises correct absence

The metric should be quality-when-triggered (3.50) + trigger-rate
(27%), not a flat mean across all events.

### Next Session

1. **Fix the eval** — update `classify_emotions.py` to report quality
   and trigger rate separately. Consider two-part eval: "Is this
   situation relevant to this drive? If yes, score 1-5." This gives
   the honest metric.

2. **Verify with filter observability** — run a short scenario with the
   LLM classifier and correlate filter logs with eval scores. Confirm
   that filtered events match the eval's low scores and rendered events
   match high scores.

3. **Test an integration test that exercises the REAL classifier** — not
   test fixture lambdas, but the actual LlmSituationClassifier against
   scenario event text. This tests the interface between components.

4. **Consider Ollama/Gemma** for local classification — eliminates API
   latency and timeout issues. The classification task is simple enough
   for a 2B-7B model.

## Branch State

- `issue-115-drive-expression-gap` (examples) — active, ~35 commits ahead of main
- `issue-115-weighted-situation-vocabulary` (neocortex) — active, revert + public constructor
- Paused branch stack: `issue-115-drive-expression-gap`

## Neocortex Commits This Session

On branch `issue-115-weighted-situation-vocabulary`:
- `e01b60a0` Revert "fix(#585): ambient+acute prompt"
- `e2d941f1` fix(#587): make LlmSituationClassifier constructor public

## Examples Commits This Session

On branch `issue-115-drive-expression-gap`:
- `c509bdd` fix(#122): appraisal-based drive filtering
- `9678f1f` test(#122): falsification tests for appraisal filtering
- `83d5824` test(#122): reverse-direction LLM eval
- `17fc2e3` fix(#122): align CharacterCognitionTest with strength-ordering + filtering
- `6b30bae` fix(#122): eval criteria — dominance not exclusivity (Principle 49)
- `0b8e731` docs(#122): Phase 8 findings — appraisal filtering implementation
- `8a7f7f1` docs(#122): scoring methodology — dominance vs exclusivity
- `b923495` data(#122): appraisal-filtering scenario run — 321 events
- `23815e6` docs(#122): scenario results — PP gallantry +0.50, mixed elsewhere
- `e108997` docs(#122): root cause — vocabulary abstraction gap
- `d5a3556` feat(#122,#587): wire LLM classifier as fallback
- `243fdf7` fix(#122): use LLM classifier directly
- `ad928d1` fix(#122): separate classifier paths — rule-based pipeline, LLM render
- `99f3d26` data(#122): LLM classifier scenario run — 306 events
- `8107fdd` feat(#122): appraisal filter observability
- `e72d74f` docs(#122): Phase 8 findings — classifier comparison, eval, observability

## Eval Data

- `wacky-manor/docs/eval/appraisal-filtering-2026-10-15/` — 321-event run, rule-based classifier
- `wacky-manor/docs/eval/llm-classifier-2026-10-15/` — 306-event run, LLM classifier
- `wacky-manor/target/experiment-results/drive-expression-eval.json` — micro test results (not committed)
