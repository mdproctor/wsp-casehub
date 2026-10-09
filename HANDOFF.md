# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Layer 1 wiring fixes complete (4 of 5 dead ends fixed), Layer 2 next
**Plan:** `.plan` has #115 active (covers=115)
**Related:** #116 — CognitivePipelineBuilder (testable pipeline assembly, not started)

## What Was Done This Session

### 1. Propagation boundary tests (TDD anchors)

Created `PropagationBoundaryTest.java` in `wacky-manor/src/test/java/.../experiment/`. Five tests, one per dead end:

| Test | Dead End | Status |
|------|----------|--------|
| driveSeeded_nodesHavePropertiesDriveAdaptationPhaseExpects | #5 | PASSES |
| formationMemories_consolidation_behavioralSectionRenders | #1 | PASSES |
| capsState_producesAgentSpecificBehavioralNodes | #3 | PASSES |
| renderCognitiveSections_includesFormationMemories | #2 | PASSES (after fix) |
| seedPeople_createsSocialSubgraphWithOverlays | #4 | PASSES |

### 2. Fixed dead end #2: renderFormationMemories uncalled

Added `renderFormationMemories()` call to `CharacterCognition.renderCognitiveSections()` when formation memories are present. "Where You Come From" section now renders between personality facets and relational schemas.

Updated `CharacterCognitionTest.allSectionsForFullyConfiguredCharacter` (7→8 sections).

### 3. Fixed dead ends #1, #3, #4 in PlaybookOrchestrator

Three wiring fixes:

| # | Dead End | Fix |
|---|----------|-----|
| 1 | `behavioralEnabled = false` | `CognitionConfig.all().with("behavioral", true)` |
| 3 | `seedCapsState()` not called | Added per-character in seeder block |
| 4 | `seedPeople()` not called | Added after character loop, before consolidation |

Added `@Inject CapsEngine capsEngine` to PlaybookOrchestrator.

### 4. Systematic dead end audit

Ran two parallel audits:
- **SocialConfig field inventory**: all 14 fields have consumers, no orphans
- **CognitionConfig subsystem audit**: 6 subsystems disabled by `all()`, `behavioralEnabled` now enabled

Discovered 2 additional dead ends beyond the original 5:
- **#6: perspectiveComparisonEnabled disabled** — cross-character view divergence not rendered
- **#7: drives invisible in solo scenes** — `resolveAdaptedDrives()` only fires with nearby agents

### 5. Created issue #116

CognitivePipelineBuilder — composable, testable pipeline assembly. Full audit details in the issue comments. Scale: M, Complexity: Low.

## Commits on Branch

```
4f3d142 docs(#115): save full-run and pipeline-hardening transcripts from 2026-10-08
65044d3 test(#115): add propagation boundary tests for 5 dead ends
42f8113 fix(#115): wire renderFormationMemories into renderCognitiveSections
fbe0262 test(#115): update CharacterCognitionTest for formation memories section
f9a683f fix(#115): wire dead ends 1/3/4 in PlaybookOrchestrator
```

## Dead End Status

| # | Dead End | Status | Fix |
|---|----------|--------|-----|
| 1 | behavioralEnabled = false | **Fixed** | CognitionConfig.with("behavioral", true) |
| 2 | renderFormationMemories uncalled | **Fixed** | Added call in renderCognitiveSections |
| 3 | seedCapsState not called | **Fixed** | Added in PlaybookOrchestrator loop |
| 4 | seedPeople not called | **Fixed** | Added after PlaybookOrchestrator loop |
| 5 | No drive rendering as somatic cues | **Open** | New prompt section needed |
| 6 | perspectiveComparisonEnabled disabled | Open (lower priority) | CognitionConfig flag |
| 7 | Drives invisible in solo scenes | Open | Separate rendering path needed |

## What Needs to Happen Next

### Immediate: Dead end #5 — Drive rendering as somatic/felt cues

This requires a new prompt section (or extension to an existing one) that renders drive intensities as somatic/felt experience — like the facet origins format: "there's a tension in your hands when you're not doing something useful" rather than "you have proving-worth drive at 0.7".

Design questions to resolve:
- New section in CharacterCognition (alongside formation memories) or new CognitionCore prompt section?
- Somatic cues derived from drive type + intensity, or from CAPS attractor output?
- How to avoid scripting — emergence constraint means drives must feel internal, not instructed

The `DriveExpressionEvalTest` somatic tests (ppAmbiguous_somaticDrives, ppAmbiguous_somaticDrivesAndMemories) already have the target format. Use those as the design reference.

### Then: Run full scenario with all wiring active

Run a GENERIC profile scenario with all 4 fixes active. Compare drive expression scores against the BASELINE0 (3.93) and Current (3.70) baselines. Save transcript to docs/eval/.

### Then: Dead end #7 — Drives in solo scenes

`resolveAdaptedDrives()` only feeds `renderSocialAwareness()` which requires nearby agents. Need a separate path that renders drive state even when alone. Could be the same somatic cue section from dead end #5.

### Then: Issue #116 — CognitivePipelineBuilder

Extract composable pipeline builder from PlaybookOrchestrator. Write three-level tests (builder, structure, propagation). Both production and test code use the same builder.

## Key Files Changed

| File | What |
|------|------|
| `wacky-manor/src/main/java/.../agent/CharacterCognition.java` | Wire renderFormationMemories into renderCognitiveSections |
| `wacky-manor/src/main/java/.../agent/PlaybookOrchestrator.java` | Enable behavioral, add seedCapsState + seedPeople, inject CapsEngine |
| `wacky-manor/src/test/java/.../experiment/PropagationBoundaryTest.java` | New — 5 boundary tests for dead ends |
| `wacky-manor/src/test/java/.../agent/CharacterCognitionTest.java` | Updated section count (7→8) |

## Architecture Notes

### Neocortex source location
The neocortex repos are at `/Users/mdproctor/claude/casehub/slots/196/neocortex`. Key files:

- `BehavioralPromptSection.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/prompt/`
  - Four-filter pipeline: subgraph type == BEHAVIORAL, CapsGenerated trait, agent-id, strength > 0.1
- `DriveAdaptationPhase.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/drive/`
  - Looks for `cognitiveKind = "drive-intensity"` in cognitive subgraphs
- `CognitionCore.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/core/`
  - `behavioralEnabled` at line 519 gates BehavioralPromptSection
- `CognitionConfig.java`: `cognition-api/src/main/java/.../core/`
  - `all()` disables 6 subsystems; `with(name, true)` enables individually
- `RelationshipStagePhase.java`: `cognition/src/main/java/.../relationship/`
  - Priority 18, evolves familiarity from stranger→confidant via interaction memories

### CognitionConfig disabled subsystems (5 remaining after enabling behavioral)

| Subsystem | What It Does | Priority |
|-----------|-------------|----------|
| perspectiveComparisonEnabled | Cross-character view divergence | Medium |
| domainActivationEnabled | Emotional pattern correlations | Low |
| salienceEnabled | Perceptual filter for appraisal | Low |
| directivePrompts | Wraps sections in behavioral directives | Quality mode |
| appraisalEnabled | Conditionally enabled by manor config | Already handled |

### Key constraint: emergence not scripting
Every change must be tested for scripting/triggering. The judge criteria require 50% weight on emergence. Drive cues must be somatic/felt, never behavioral directives. A scripted 5/5 is worth 2/5 at most.

## Baseline Transcripts — DO NOT DELETE

These transcripts are the comparison baselines for the #97 iterative pare-back. They live in `wacky-manor/docs/eval/` on the branch and main:

| Transcript | Events | What it tests | Profile |
|---|---|---|---|
| `old-briefings-20261002/transcript.json` | 327 | Verbose briefings (pre-rewrite) | BASELINE |
| `new-briefings-20261002/transcript.json` | 333 | Directive-minimal (post-rewrite) | BASELINE |
| `generic-briefings-20261002/transcript.json` | 306 | Generic + prescribed catchphrases | GENERIC |
| `generic-colloquial-20261002/transcript.json` | 310 | Generic + style-directive only | GENERIC |
| `full-run-2026-10-08.json` | — | Full run baseline | — |
| `pipeline-hardening-run-2026-10-08/transcript.json` | — | Pipeline hardening run | — |

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```

**Run tests:**
```bash
# Standard tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -s .mvn/slot-settings.xml

# Boundary tests only
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Dtest="PropagationBoundaryTest" -s .mvn/slot-settings.xml

# Drive expression eval tests (LLM required)
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -Dtest="DriveExpressionEvalTest" -s .mvn/slot-settings.xml
```

## Pre-existing test failures (not caused by #115)

- `RelationshipStageIntegrationTest` (3 errors) — IllegalAccessError on package-private constructor
- `BeliefRevisionIntegrationTest` (3 errors) — same issue, neocortex API visibility change

## Branch State

- `issue-115-drive-expression-gap` — active, 8 commits (5 this session), Layer 1 complete
- Project repo and workspace repo both on this branch
