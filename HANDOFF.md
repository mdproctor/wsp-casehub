# HANDOFF — casehub-examples (slot 196)

## Current Work

**Branch:** `main` (elastic session scheduler epic closed)
**Next:** neocortex#661 — Replace decay-based consolidation trigger with monotonic divergence accumulation
**Plan:** `.plan` scaffolded with #661 as first priority

## START HERE — What's Next

Run `work start` to begin neocortex#661. Branch: `issue-661-monotonic-consolidation-trigger`.

The issue has full literature citations and a three-part design:
1. **Event count** (monotonic, no decay) — Park et al.'s baseline. Fire at 5-10 un-processed interactions.
2. **Divergence accumulation** (monotonic, weighted by novelty) — CLS 2016's key insight. Weight by appraisal divergence from consolidated model.
3. **Time ceiling** — existing `interval-ticks`, unchanged.

Key constraint: `DegreeAccumulator` stays valid for CAPS classification (detecting behavioural shifts). Only consolidation scheduling gets the new mechanism.

Literature review: `wacky-manor/docs/research/consolidation-trigger-literature-2026-10-22.md`
Blog entry: `wsp-casehub/blog/2026-10-22-mdp01-consolidation-trigger-decay.md`
Parameter tuning report: `wacky-manor/docs/eval/tuned-run-2026-10-22/PARAMETER-TUNING.md`

## What was completed this session

### Elastic session scheduler epic — CLOSED (15 issues)

**examples#138 — Scenario run and parameter tuning (complete)**
- Ran baseline + tuned 50-tick GENERIC scenarios with all cognitive components
- Identified and fixed: session pool starvation, disabled appraisal, 4h mood decay, accumulator thresholds
- Mood dynamic range improved 66x; character differentiation working
- Externalized mood config, accumulator thresholds, appraisal toggle

**platform#655 — Pool semaphore decoupled from session lifecycle (complete)**
- Semaphore removed from SessionPool — session creation now free
- AIMD controls active prompt execution (the expensive resource)
- Picked up from canonical main, rebased slot repos

**platform#650, #652, #653 — Epic issues closed**
- Branch squashed (14→7 commits), fast-forward merged to main, stamped

### Design discovery: consolidation trigger decay is wrong

Found during parameter tuning — character drives static across 50 ticks because CUSUM decay suppresses slow-building personality changes. Literature review confirmed: Park et al. (2023), SCM (Shinde 2026), CLS theory (McClelland/Kumaran 2016) all converge on monotonic accumulation with no decay. Filed as neocortex#661.

## Previous Session

Fixed the drive expression pipeline end-to-end: model routing,
appraisal filtering, and profile derivation.

**Appraisal filter redesign (examples repo):**
- Binary overlap (`> 0`) → relative overlap (`overlap/profileTotal >= 0.15`)
- `context: social` field added then REMOVED — replaced by topology-inferred gating
- `resolveAdaptedDrives()` preserves context/triggers from original config

**Discriminative profile derivation (neocortex repo):**
- `DriveProfileDerivationPhase.deriveProfiles()` passes discrimination metadata
- Classifier sees other drives as contrast: "return features distinctive to
  this drive, not shared with the others"
- First attempt: gallantry narrowed 7→5 nodes, acceptance became top node,
  but physical_threat and competence_recognition leaked through

**Test harness:**
- `DriveRenderingUnitTest` — 13-scenario cross-matrix, 3 characters,
  solo/social/competing-signals, 0.2s. Fixed profiles, mock classifiers.
- `DriveProfilePipelineTest` — real LLM derivation + classification,
  PP + HC, ~10min. Scoped to tested agents only.
- `ClassifierModelComparisonTest` — diagnostic: haiku vs opus activations

**Documentation:**
- Principles 59-64 in CHARACTER-EMERGENCE-FINDINGS.md
- Blog: "Why Every Drive Looked the Same (and What Fixed It)"

### Current State

**Architecture proven, prompt needs iteration.**

Unit test matrix: 13/13 pass with discriminative profiles (hand-curated
to represent what good derivation output looks like).

Pipeline test: 4/8 pass with LLM-derived discriminative profiles.
The discrimination metadata improved output but didn't fully constrain it.
Gallantry still gets physical_threat:0.850 and competence_recognition:0.750
alongside the correct acceptance:0.800.

### Immediate Next Step

Iterate on the discriminative derivation prompt in neocortex
`DriveProfileDerivationPhase.buildDiscriminationMetadata()`. Options:

1. **Constrain output count** — "classify into AT MOST 4-5 input nodes"
2. **Provide explicit contrast** — "competence_recognition is NOT
   distinctive to gallantry because proving-worth also has it"
3. **Two-pass derivation** — first pass gets raw profile, second pass
   prunes shared nodes using IDF-style discrimination

Test with `DriveProfilePipelineTest` (PP + HC, ~10min). Target: 8/8.

### What NOT to do

- Don't re-add `context: social` as an authored field — social gating
  should be inferred from profile category concentration
- Don't add authored triggers — the system should be fully emergent
- Don't run all 17 characters in the pipeline test — scope to tested agents

## Branch State

- `issue-115-drive-expression-gap` (examples) — active, 12 commits this session
- `issue-115-weighted-situation-vocabulary` (neocortex) — active, 1 commit this session
- `fix/claude-agent-model-passthrough` (platform) — active, 3 commits

## Key Files

- `examples/wacky-manor/.../DriveRenderingUnitTest.java` — fast cross-matrix
- `examples/wacky-manor/.../DriveProfilePipelineTest.java` — full LLM pipeline
- `examples/wacky-manor/.../CharacterCognition.java` — appraisalFilterDrives, computeInputOverlap
- `neocortex/mindmap-intelligence/.../DriveProfileDerivationPhase.java` — discriminative derivation
- `platform/agent-claude-core/.../ClaudeAgentClient.java` — model passthrough
- `examples/wacky-manor/docs/CHARACTER-EMERGENCE-FINDINGS.md` — Principles 59-64
- `wsp-casehub/blog/2026-10-16-mdp01-discriminative-drive-profiles.md` — diary entry

## Platform Commits

On `fix/claude-agent-model-passthrough`:
- `f69aed35` fix(#589): pass model from AgentSessionConfig to CLI options
- `e3236da7` test(#589): haiku alias resolves to FAST-tier claude model
- `531d4d31` fix(#589): pass unrecognized model strings to default backend

## Neocortex Commits

On `issue-115-weighted-situation-vocabulary`:
- `0c69e4df` fix(#589): discriminative profile derivation — contrast other drives

## Stashed Work

`authored-triggers-wip` stash in examples — the triggers approach that
was abandoned in favour of emergent inference. Drop it.
