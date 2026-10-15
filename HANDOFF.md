# HANDOFF — casehub-examples

## Last Session

Fixed the eval methodology — two-part relevance gating separates drive
quality from trigger frequency. Ran the integration test that exercises
the REAL LlmSituationClassifier against scenario text.

### What was done

**Eval methodology fix (Principle 53):**
- `classify_emotions.py` updated with two modes: `flat` (original) and
  `relevance` (new default). Relevance mode asks: "Is this drive
  relevant to this situation? If yes, score 1-5."
- Reports quality-when-triggered and trigger-rate as independent metrics
- Re-classified 215 events from the LLM classifier transcript

**Relevance eval results (key numbers):**
- PP gallantry: quality 2.65 at 72% trigger rate (B0 was 4.50 via contamination)
- HC scheming: quality 4.70 at 100% trigger rate (always-on proactive drive)
- HC gloating: quality 2.50 at 28% trigger rate (genuinely reactive)
- Pen curiosity: quality 4.55 at 100% trigger rate (always-on)
- Pen adventure: quality 2.93 at 71% trigger rate (situational)

**Integration test — SituationClassifierIntegrationTest (7/7 pass):**
- Exercises REAL `LlmSituationClassifier` with real `CapsTopology`
- Covers: social warmth, physical struggle, competence, investigation,
  competition, neutral, and cross-scenario differentiation
- Retry logic for LLM non-determinism (up to 3 attempts for empty results)

**Design principles established (Principles 53-54):**
- #53: Eval must use two-part relevance gating before scoring expression
- #54: Classifier uses subject's experiential perspective, not observer's
  appraisal — DriveProfileInferred inputs must match this perspective

**Integration test finding (Principle 54):**
The classifier classifies from the SUBJECT's experiential perspective:
- "Clara struggling with door" → `powerlessness/failure/physical_threat`
  (Clara's experience)
- "Clara welcomes you warmly" → `acceptance/secure_attachment`
  (social warmth — the gallantry territory)
This means gallantry renders only in genuinely warm social interactions,
not in physical distress scenarios. Protection renders for distress.
The distinction is architecturally correct but DriveProfileInferred
inputs must be structured to match the classifier's perspective.

### What was NOT done

**Priority 2 from previous session** — filter observability correlation
(run a short scenario, correlate `[appraisal-filter]` log lines with
per-event eval scores). Not attempted this session — the relevance eval
already provides the honest metric. This remains useful for debugging
false negatives but is no longer the critical path.

**Ollama/Gemma exploration** — local classification for latency reduction.
Deferred.

### Next Session

1. **Address the gallantry quality gap** — quality 2.65 when triggered
   is below the 3.0 "clearly present" threshold. The issue isn't trigger
   frequency (fixed by relevance eval) but expression quality even in
   relevant situations. Investigate: are the multi-channel descriptions
   too abstract? Does the drive intensity (0.9) translate to strong
   expression?

2. **Consider filter observability correlation** — run a short scenario,
   capture `[appraisal-filter]` log lines, cross-reference with relevance
   eval scores. Confirm rendered events score 3+ and filtered events
   would have scored 1-2. If a rendered event scores 1, the description
   is the problem. If a filtered event would have scored 3+, the filter
   is too aggressive.

3. **Establish BASELINE1** — the four regressions from L1 emotional echo
   (PP protection -1.13, DD gloating -0.48, DD recognition -0.37,
   Pen social-harmony -0.33) still need recovery. Richer drive
   descriptions, intensity tuning, or selective tendency restoration.

## Branch State

- `issue-115-drive-expression-gap` (examples) — active, ~40 commits ahead of main
- `issue-115-weighted-situation-vocabulary` (neocortex) — active, revert + public constructor
- Paused branch stack: `issue-115-drive-expression-gap`

## Examples Commits This Session

On branch `issue-115-drive-expression-gap`:
- `98d32b6` feat(#122): two-part eval — relevance gate separates quality from trigger rate
- `3585f85` test(#122): integration test — real LlmSituationClassifier against scenario text
- `b410471` data(#122): relevance eval results + integration test revision
- `81efe3c` docs(#122): Phase 8 findings — relevance eval results, Principles 53-54

## Eval Data

- `wacky-manor/docs/eval/appraisal-filtering-2026-10-15/` — 321-event run, rule-based classifier
- `wacky-manor/docs/eval/llm-classifier-2026-10-15/` — 306-event run, LLM classifier
- `wacky-manor/docs/eval/llm-classifier-2026-10-15/relevance-results.json` — relevance eval (215 events)
- `wacky-manor/target/experiment-results/situation-classifier-*.json` — integration test results (not committed)
