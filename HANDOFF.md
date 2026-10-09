# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**Design:** casehubio/examples#117 — Unified disposition renderer
**State:** Unified renderer implemented, tests pending
**Plan:** `.plan` has #115 active (covers=115), #117 queued
**Related:** #116 — CognitivePipelineBuilder (testable pipeline assembly, not started)

## What Was Done This Session

### 1. Root-to-tip pipeline analysis (four parallel forks)

Traced the full drive expression pipeline. Key finding: three disconnected drive systems — orchestrator drives (cognitive engagement, works), character drives (personality disposition, suppressed), CAPS attractors (emergent behavior, works). Character-specific drives have zero rendering path because `CharacterDrivePromptSection` is gated by `!appraisalActive` and manor enables appraisal.

### 2. Created design issue #117

Full analysis documented: pipeline traces, gap inventory, eval findings, unified renderer design, test strategy. Cross-referenced from #115.

### 3. Unified renderInternalState() implemented

New method in `CharacterCognition` that reads BOTH:
- Character drives from cognitive subgraph (via `resolveAdaptedDrives()`)
- CAPS behavioral attractors from BEHAVIORAL subgraph

Renders a single "What Pulls at You" section in somatic format. Positioned after "Who You Are", before "Where You Come From". No nearby-agents gate — works in both solo and social scenes.

### 4. BehavioralPromptSection disabled

Removed `.with("behavioral", true)` from PlaybookOrchestrator CognitionConfig. CAPS attractors now render through the unified renderer instead of BehavioralPromptSection.

### 5. PP drive descriptions rewritten to somatic format

Both BASELINE and GENERIC profiles. Analytical descriptions ("Must protect and impress Penelope") replaced with somatic felt-state text ("A warmth moves through you when courtesy matters...").

### 6. Tests updated

- CharacterCognitionTest: 8→9 sections with "What Pulls at You"
- PropagationBoundaryTest: two new tests for unified renderer (solo + full pipeline)
- Test run in progress

## Commits on Branch

```
e30fe7a test(#115): update section count and add boundary tests for unified renderer
169b093 wip(#115): unified renderInternalState — somatic drives + CAPS attractors
f9a683f fix(#115): wire dead ends 1/3/4 in PlaybookOrchestrator
fbe0262 test(#115): update CharacterCognitionTest for formation memories section
42f8113 fix(#115): wire renderFormationMemories into renderCognitiveSections
65044d3 test(#115): add propagation boundary tests for 5 dead ends
4f3d142 docs(#115): save full-run and pipeline-hardening transcripts from 2026-10-08
```

## Dead End Status

| # | Dead End | Status | Fix |
|---|----------|--------|-----|
| 1 | behavioralEnabled = false | **Fixed** → **Superseded** | Unified renderer replaces BehavioralPromptSection |
| 2 | renderFormationMemories uncalled | **Fixed** | Added call in renderCognitiveSections |
| 3 | seedCapsState not called | **Fixed** | Added in PlaybookOrchestrator loop |
| 4 | seedPeople not called | **Fixed** | Added after PlaybookOrchestrator loop |
| 5 | No drive rendering as somatic cues | **Fixed** | renderInternalState() with somatic descriptions |
| 6 | perspectiveComparisonEnabled disabled | Open (lower priority) | Separate scope |
| 7 | Drives invisible in solo scenes | **Fixed** | Same unified renderer (no nearby-agents gate) |

## What Needs to Happen Next

### Immediate: Verify tests pass

Tests are running. If CharacterCognitionTest or PropagationBoundaryTest fail, debug and fix.

### Then: Run somatic eval tests

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -Dtest="DriveExpressionEvalTest#ppAmbiguous_somaticDrives_provingWorth,DriveExpressionEvalTest#ppAmbiguous_somaticDrivesAndMemories_provingWorth" -s .mvn/slot-settings.xml
```

Target: ≥3/5 on both expression and emergence dimensions.

### Then: Run full scenario with all wiring active

Run a GENERIC profile scenario with all fixes active. Compare drive expression scores against BASELINE0 (3.93). Save transcript.

### Then: Rewrite remaining characters' drive descriptions

If PP validates, rewrite drives for: hooded-claw, penelope-pitstop, dick-dastardly, ant-hill-mob.

### Then: Consider #116 CognitivePipelineBuilder

Extract composable pipeline builder from PlaybookOrchestrator. Separate scope.

## Key Files Changed This Session

| File | What |
|------|------|
| `CharacterCognition.java` | Added renderInternalState() + renderAttractorAsFelt() + wiring |
| `PlaybookOrchestrator.java` | Disabled BehavioralPromptSection via CognitionConfig |
| `social-config.yaml` | PP drive descriptions → somatic format |
| `social-config-generic.yaml` | PP drive descriptions → somatic format |
| `CharacterCognitionTest.java` | 8→9 sections, "What Pulls at You" in expected order |
| `PropagationBoundaryTest.java` | Two new boundary tests for unified renderer |

## Architecture Notes

### Neocortex source location
The neocortex repos are at `/Users/mdproctor/claude/casehub/slots/196/neocortex`. Key files:

- `BehavioralPromptSection.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/prompt/`
  - Four-filter pipeline: subgraph type == BEHAVIORAL, CapsGenerated trait, agent-id, strength > 0.1
  - NOW DISABLED — unified renderer in CharacterCognition reads same data
- `DriveAdaptationPhase.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/drive/`
  - Looks for `cognitiveKind = "drive-intensity"` in cognitive subgraphs
- `CognitionCore.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/core/`
  - `behavioralEnabled` now false (unified renderer replaces it)
- `CharacterDrivePromptSection.java`: suppressed by `!appraisalActive` — not used
- `CognitionConfig.java`: `cognition-api/src/main/java/.../core/`
  - `all()` disables subsystems; `with(name, true)` enables individually

### Three drive systems (documented in #117)

| System | Level | Renderer | Status |
|--------|-------|----------|--------|
| Orchestrator drives | Cognitive engagement | DrivePromptSection | Works (separate) |
| Character drives | Personality disposition | **renderInternalState()** | **NEW — unified** |
| CAPS attractors | Emergent behavior | **renderInternalState()** | **NEW — unified** |

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

- `issue-115-drive-expression-gap` — active, 12 commits (7 this session), Layer 1 + Layer 2 complete
- Project repo and workspace repo both on this branch
