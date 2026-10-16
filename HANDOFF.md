# HANDOFF — casehub-examples

## Last Session

Deep investigation into appraisal filtering pipeline. Found and fixed
multiple bugs. Built test harness for compositional drive testing.
Blocked on AgentProvider model routing — classifier was using Opus
instead of Haiku for simple JSON classification.

### What was done

**Eval methodology fix (Principle 53):**
- `classify_emotions.py` two-part relevance mode (default)
- Re-classified 215 events: PP gallantry quality 2.65 at 72% trigger
- Proactive drives (scheming, curiosity) 100% trigger — always-on confirmed

**Root cause chain (neocortex#589):**
1. Rule-based classifier keyword-matches narrative memory text → wrong
   profile inputs (gallantry got `dominance:0.80` from "bullied")
2. Disabled rule-based, wired LLM classifier for consolidation
3. LLM classifier was accidentally using Opus (no model specified in
   AgentSessionConfig) → 20-30s per call, frequent timeouts
4. Added fail-fast (ClassificationException) + resumable derivation
5. Added drive description classification alongside memories
6. Added relationship metadata to classify calls
7. Fixed model to haiku — but AgentProvider isn't routing it correctly

**Pipeline test harness (DriveProfilePipelineTest):**
- Parameterized: seed all characters once, run scenarios as data rows
- Tests full chain: seed → derive profiles → classify situation → filter → assert
- 5 PP gallantry scenarios defined (social, solo, mixed, danger)
- Currently blocked by classifier timeouts — model routing needs investigation

**Reproducer tests (SituationClassifierIntegrationTest):**
- Real scenario event text classification
- Fix candidate comparison (richer text vs metadata vs threshold)
- Metadata with relationship context doubles confidence (0.30→0.50)

**Key unknown: which model was used for previous baselines?**
- B0: no classifier
- LLM classifier runs: accidentally Opus (not Haiku as assumed)
- Python eval: explicitly Haiku via Vertex
- This means scenario run results and eval results used DIFFERENT models

### Current Failure to Reproduce

**Test:** `DriveProfilePipelineTest` (parameterized pipeline harness)
**Run:** `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -Dtest="DriveProfilePipelineTest" -s .mvn/slot-settings.xml`
**What happens:** `ClassificationException` during `@BeforeAll` profile derivation. The `LlmSituationClassifier` times out on formation memory classification. 3 retry attempts all fail.
**Root cause:** `AgentSessionConfig.withModel("haiku")` is not routing to Haiku. The CLI still uses Opus (confirmed by stderr: "Opus 5.5 not available — using Opus 4.6"). Opus takes 20-30s per classification call; the 30s timeout is too tight.

**The deeper failure we haven't reproduced as a red test:**
In the full scenario run (run4, `/tmp/wacky-manor-run4.log` if server still running), gallantry was filtered in 83% of PP events. The dominant situation activation was `modeling_observed:0.70` — the classifier returns this for almost every social interaction event. Gallantry's profile doesn't include `modeling_observed`, so no overlap, gallantry filtered.

We wrote tests for this (`SituationClassifierIntegrationTest` reproducer scenarios, `DriveProfilePipelineTest`) but they passed because they ran separate LLM derivation runs that produced different (better) profile inputs than the scenario. **The tests don't reproduce the actual failure because profile derivation is non-deterministic across LLM runs.**

**To make progress, the next session must:**
1. Determine what model string routes to Haiku through the AgentProvider. Check `ClaudeAgentProvider` or `BackendInstanceCoordinator` in the platform repo. The classifier is in `neocortex/caps-llm-classifier/.../LlmSituationClassifier.java` line 78.
2. Once Haiku works: run `DriveProfilePipelineTest` — it should pass reliably (Haiku is fast, no timeouts).
3. Then check: does the test FAIL if you revert the metadata fix (`buildSituationMetadata` in `CharacterCognition`)? If not, the test doesn't cover the bug.
4. Separately: consider whether LLM non-determinism in profile derivation means the pipeline needs Ollama (deterministic, local, fast) rather than an API-based model.

**Key files:**
- `neocortex/caps-llm-classifier/.../LlmSituationClassifier.java` — classifier, model selection (line 78)
- `examples/wacky-manor/.../CharacterCognition.java` — `appraisalFilterDrives`, `buildSituationMetadata`
- `examples/wacky-manor/.../DriveProfilePipelineTest.java` — parameterized test harness
- `examples/wacky-manor/.../SituationClassifierIntegrationTest.java` — reproducer + fix candidate tests
- `neocortex/mindmap-intelligence/.../DriveProfileDerivationPhase.java` — profile derivation with description classification
- `examples/wacky-manor/.../CapsProducer.java` — CDI bean producing LlmSituationClassifier

## Branch State

- `issue-115-drive-expression-gap` (examples) — active
- `issue-115-weighted-situation-vocabulary` (neocortex) — active

## Neocortex Commits This Session

On branch `issue-115-weighted-situation-vocabulary`:
- `c121607e` fix(#589): disable RuleBasedSituationClassifier
- `65bd9d08` chore(#589): label commit
- `2ccc7f8b` fix(#589): fail fast on classification timeout
- `3e63bfba` fix(#589): resumable drive profile derivation
- `934897f1` feat(#589): classify drive description alongside memories
- `8cdcb5d5` fix(#589): make DriveProfileDerivationPhase constructor public
- `cbd814c9` fix(#589): use haiku model for classification

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

## Neocortex Commits This Session

On branch `issue-115-weighted-situation-vocabulary`:
- `c121607e` fix(#587): disable RuleBasedSituationClassifier — keyword matching produces wrong drive profiles

## Examples Commits This Session

On branch `issue-115-drive-expression-gap`:
- `98d32b6` feat(#122): two-part eval — relevance gate separates quality from trigger rate
- `3585f85` test(#122): integration test — real LlmSituationClassifier against scenario text
- `b410471` data(#122): relevance eval results + integration test revision
- `81efe3c` docs(#122): Phase 8 findings — relevance eval results, Principles 53-54
- `266331f` fix(#122): wire LlmSituationClassifier as CDI SituationClassifier

## Eval Data

- `wacky-manor/docs/eval/appraisal-filtering-2026-10-15/` — 321-event run, rule-based classifier
- `wacky-manor/docs/eval/llm-classifier-2026-10-15/` — 306-event run, LLM classifier
- `wacky-manor/docs/eval/llm-classifier-2026-10-15/relevance-results.json` — relevance eval (215 events)
- `wacky-manor/target/experiment-results/situation-classifier-*.json` — integration test results (not committed)
