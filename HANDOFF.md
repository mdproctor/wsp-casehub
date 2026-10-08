# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Diagnostic complete, implementation not started
**Plan:** `.plan` has #115 active (covers=115)

## What Was Done This Session

### 1. Established the problem scope

Drive expression eval shows secondary/performative drives (gloating, gallantry, proving-worth) collapse without prescriptive text. Overall mean drops BASELINE0 3.93 → Current 3.70. Eight drives across five characters affected.

### 2. Wrote micro tests for PP proving-worth

Created `DriveExpressionEvalTest.java` in `wacky-manor/src/test/java/.../experiment/`. Tests:

- **ppDiagnostic_dumpCognitiveContext** — dumps what PP currently sees in the prompt. Reveals drives not rendered, formation memories not in prompt, only facet origins surfaced.
- **ppProvingWorth_competitorPraised_showsDrivenResponse** — direct trigger test. Scored **5/5** — proves LLM can express proving-worth.
- **ppAmbiguous_noTrigger_baselineProvingWorth** — alone in library, no trigger. Scored **3/5** baseline.
- **ppAmbiguous_drivesInjected_provingWorth** — behavioral drive text injected. Scored **4/5**.
- **ppAmbiguous_memoriesInjected_provingWorth** — enriched memories injected. Scored **3/5**.
- **ppAmbiguous_drivesAndMemories_provingWorth** — both injected. Scored **5/5**.
- **ppAmbiguous_somaticDrives_provingWorth** — somatic/felt drive cues (not yet run).
- **ppAmbiguous_somaticDrivesAndMemories_provingWorth** — somatic + memories (not yet run).

Judge criteria updated to require BOTH expression AND emergence (50/50 weight). A scripted 5 on expression is worth 2 at most.

### 3. Root-to-tip propagation analysis

Traced the full pipeline from YAML personality data to LLM prose. Found **five dead ends** where personality data is processed but never reaches the prompt:

| # | Dead End | Root Cause | Location |
|---|---------|-----------|----------|
| 1 | BehavioralPromptSection renders null | `behavioralEnabled = false` in `CognitionConfig.all()` | neocortex `CognitionCore.java` line 519 |
| 2 | renderFormationMemories() never called | Method exists at CharacterCognition line 210, zero callers | wacky-manor `CharacterCognition.java` |
| 3 | seedCapsState() never called in production | Only called in tests | wacky-manor `ManorCognitiveSeeder.java` |
| 4 | seedPeople() never called in production | Only called in tests | wacky-manor `ManorCognitiveSeeder.java` |
| 5 | No drive rendering path | Drives seeded as MindMap nodes but no section surfaces them | Gap — path doesn't exist |

Of 14 personality data types, only 5 reach the LLM. The richest ones (AMPD scores, memories, drives, CAPS attractors, disposition) all hit dead ends.

### 4. Full write-up in issue #115

Issue body updated with complete diagnostic: propagation graph, data flow diagram, all five dead ends with locations, experimental evidence table, three-layer fix architecture, and collapsed drive root cause mapping.

Two comments also on the issue with supplementary analysis.

## Key Files Changed

| File | What |
|------|------|
| `wacky-manor/src/test/java/.../experiment/DriveExpressionEvalTest.java` | New — micro tests for drive expression gaps |
| `CLAUDE.md` | Updated with eval framework documentation (from #109 session) |

## Commits on Branch

```
1c4b078 test(#115): add drive expression eval — PP proving-worth baseline
d1311ae test(#115): hypothesis tests — drives vs memories vs combined
7660fb5 wip: add somatic + emergence criteria to drive expression tests
```

## What Needs to Happen Next

### Immediate: TDD the propagation boundary points

Each dead end needs an isolated unit test proving the signal crosses that boundary. Write these BEFORE making any wiring changes:

1. **Test: drives seeded → DriveAdaptationPhase finds them**
   - Create isolated MindMap, seed drives via `ManorCognitiveSeeder.seed()`, run `DriveAdaptationPhase`, assert drive intensities are found and can be adapted
   - Investigate why "No drive nodes found" appears despite `seed()` creating nodes with `cognitiveKind: "drive-intensity"` property

2. **Test: formation memories → BehavioralPromptSection renders**
   - Seed formation memories, run consolidation (produces 50 CAPS attractor nodes), enable `behavioralEnabled = true`, call `BehavioralPromptSection.render()`, assert non-null output
   - Check all four filters: subgraph type == "behavioral", trait == "CapsGenerated", agent-id match, strength > 0.1

3. **Test: seedCapsState → CAPS attractors reflect disposition**
   - Call `seedCapsState()` with PP's disposition, run CAPS settling, assert attractors differ from non-disposition baseline

4. **Test: renderFormationMemories produces section**
   - Call the method, assert it returns an `ObservationSection` with memory content

5. **Test: seedPeople → RelationshipStagePhase evolves**
   - Call `seedPeople()`, add interaction memories, run `RelationshipStagePhase`, assert familiarity stage advanced

### Then: Layer 1 wiring fixes

Once boundary tests pass, make the wiring changes:

1. Enable `behavioralEnabled` in PlaybookOrchestrator's CognitionConfig
2. Call `renderFormationMemories()` from `renderCognitiveSections()`
3. Call `seedCapsState()` from PlaybookOrchestrator per-character setup
4. Call `seedPeople()` from PlaybookOrchestrator before consolidation
5. Fix GENERIC formation memory PAD values (all currently 0.0)

### Then: Layer 2 inference enrichment

6. Add drive rendering as somatic/felt cues (not prescriptive — every change tested for scripting vs emergence)
7. Enhance CAPS attractor output — personality-specific texture, not generic one-word labels
8. Derive expression styles from AMPD trait constellations
9. Connect appraisal to prompt section selection — situation-dependent foregrounding

### Then: Layer 3 data enrichment

10. Add formation memories for DD, Penelope, AHM (currently zero)
11. Add AMPD facets for DD (currently zero)
12. Enrich existing memories with expression cues

## Architecture Notes

### Neocortex source location
The neocortex repos are at `/Users/mdproctor/claude/casehub/slots/196/neocortex`. Key files found by exploration agents:

- `BehavioralPromptSection.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/prompt/`
  - Four-filter pipeline: subgraph type, CapsGenerated trait, agent-id, strength > 0.1
  - Returns null if no nodes pass all filters
- `DriveAdaptationPhase.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/drive/`
  - Looks for `cognitiveKind = "drive-intensity"` in cognitive subgraphs
  - Reads `agent-id`, `drive-type`, `intensity` properties
- `CognitionCore.java`: `cognition/src/main/java/io/casehub/neocortex/cognition/core/`
  - `CognitionConfig.all()` line 28: behavioral is the 22nd param, set to `false`
  - Only `CognitionConfig.withAllSubsystems()` enables it
- `BehavioralSynthesisPhase.java`: `mindmap-intelligence/src/main/java/.../consolidation/`
  - Creates "behavioral" type subgraph, adds nodes with "CapsGenerated" trait
  - Sets properties: caps-node-id, category, strength, agent-id, generation

### PlaybookOrchestrator setup chain
Per-character setup in `runScenario()`:
1. Look up AgentDescriptor from AgentRegistry
2. Load SocialConfig from ManorSocialConfigLoader.loadForProfile(profileMode)
3. Derive CognitiveDefaults via ManorCognitiveSetup.deriveDefaults()
4. `seeder.seed()` — beliefs + drives + needs into MindMap
5. `seeder.seedGoals()` — drive-aligned goals into goal orchestrator
6. `seeder.seedFormationMemories()` — formative experiences into memory
7. **NOT called:** `seedCapsState()`, `seedPeople()`
8. Construct CharacterCognition

### Key constraint: emergence not scripting
Every change must be tested for scripting/triggering. The judge criteria require 50% weight on emergence. Drive cues must be somatic/felt (like facet origins: "there's a tension in your hands when you're not doing something useful"), never behavioral directives ("you have a restless need to demonstrate capability"). A scripted 5/5 is worth 2/5 at most.

## Baseline Transcripts — DO NOT DELETE

These transcripts are the comparison baselines for the #97 iterative pare-back. Every future ablation run compares against these. They live in `wacky-manor/docs/eval/` on main:

| Transcript | Events | What it tests | Profile |
|---|---|---|---|
| `old-briefings-20261002/transcript.json` | 327 | Verbose briefings (pre-rewrite) — behavioral ceiling | BASELINE |
| `new-briefings-20261002/transcript.json` | 333 | Directive-minimal (post-rewrite) — parity confirmation | BASELINE |
| `generic-briefings-20261002/transcript.json` | 306 | Generic chars + prescribed catchphrases — taxonomy validation | GENERIC |
| `generic-colloquial-20261002/transcript.json` | 310 | Generic chars + style-directive only — emergence validation | GENERIC |
| `directive-minimal-20261002/partial-events.json` | 25 | Partial earlier run (kept for historical reference) | COMPOSITE |

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```

**Run tests:**
```bash
# Standard tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -s .mvn/slot-settings.xml

# Drive expression eval tests
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -Dtest="DriveExpressionEvalTest" -s .mvn/slot-settings.xml

# Pipeline output diagnostic
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pllm-eval -Dtest="PipelineOutputTest" -s .mvn/slot-settings.xml
```

## Branch State

- `issue-115-drive-expression-gap` — active, 3 commits, diagnostic tests only (no production changes)
- Project repo and workspace repo both on this branch
