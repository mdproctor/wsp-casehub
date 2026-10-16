# HANDOFF — casehub-examples

## Last Session

Fixed the drive expression pipeline end-to-end: model routing,
appraisal filtering, and profile derivation. Arrived at a compositional
solution through five iterations.

### What was done

**Model routing (platform repo):**
- `ClaudeAgentClient.buildCliOptions()` now passes `config.model()` to CLI
- `RoutingAgentProvider.resolve()` passes unrecognized model strings to default backend
- `withModel("haiku")` now reaches the Claude CLI as `--model haiku`

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
