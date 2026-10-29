# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-29
**Branch:** wip/era5-relational-eval (examples), main (neocortex)
**Era:** 5 — relational cognition, targeted somatic drives
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## Last Session

Re-architected the cognitive pipeline to produce targeted somatic drives. The problem: Era 5's relational schemas (#477) degraded somatic register from 3.74 to 3.13 when added as separate prompt sections. The diagnosis went through three layers:

1. **Redundancy** — relational info scattered across 4+ sections
2. **Instruction competition** — cognitive-analytical sections ("Prey. Trust 0.1.") gave the LLM an easier register than Barrett somatic, and it took the easier path
3. **Phenomenological category error** — drives and targets rendered as independent dimensions, forcing LLM cross-reference under attention pressure

**Critical design correction:** Initial approach was removing cognitive sections (relational schemas, attachment, trust). User corrected: those sections represent cognitive abilities — removing them lobotomizes the character. The fix is better wiring (targeted prose + correct appraisal), not less cognition.

**Key technical finding:** `DriveDescriptionGenerationPhase` (P22) received zero relational context despite running AFTER `RelationalSchemaEvolutionPhase` (P20). Somatic descriptions were personality-grounded but completely target-agnostic. `OtherModelDerivationPhase` (P25) already demonstrated the pattern of reading schemas during consolidation — P22 just didn't do it.

**Additional finding:** `renderRelationalSchemas()` in CharacterCognition was reading static YAML config, never the evolved overlay data from P20. Relationship evolution was computed but invisible to the prompt.

## What Was Built (all tests green)

| # | Component | Repo | What |
|---|-----------|------|------|
| 1 | `DriveDescriptionGenerationPhase` | neocortex/mindmap-intelligence | Reads relational schemas, generates per-drive-per-role Barrett descriptions (21 tests) |
| 2 | `TargetedDrive` record | neocortex/cognition | Drive + optional target agent/display name/role |
| 3 | `resolveRelationalTargets()` | neocortex/cognition | Scene-level drive→target via overlay schemas + drive-role salience (10 tests) |
| 4 | `renderInternalState()` 5-arg overload | neocortex/cognition | Renders `DRIVE (0.7) → Penelope: [targeted description]` |
| 5 | `readGeneratedDriveDescriptions()` | neocortex/cognition | Returns role-keyed entries alongside untargeted |
| 6 | `renderRelationalSchemas()` | examples/wacky-manor | Reads evolved overlay data, YAML fallback (46 tests) |
| 7 | Wiring | examples/wacky-manor | `renderCognitiveSections` calls 5-arg overload with `agentNames` |

Pipeline is connected end-to-end. No eval has been run yet.

## What Worked

- **Per-drive-per-role generation** — one LLM call per drive (same as before), schemas as context. LLM returns `{"description": "...", "targeted": {"role": "..."}}` and both are stored. Idempotent rerun skips existing descriptions.
- **Backward-compatible overload** — 5 callers of `renderInternalState` untouched. New 5-arg overload adds targeting. Old 4-arg delegates with empty map.
- **Drive-role salience** — `computeDriveRoleSalience()` switch statement maps drive types to compatible schema roles. Deterministic, no LLM needed at render time. Returns 0.0 for non-social drives (curiosity, adventure), leaving them untargeted.

## What Needs Scrutiny

- **`computeDriveRoleSalience` is hardcoded** — maps gloating→prey/rival, gallantry→beloved/ward, etc. Works for current character set but is brittle. May need to become data-driven or LLM-resolved during consolidation.
- **`parseTargeted` uses regex on LLM JSON** — no JSON library, consistent with existing `parseField` approach. Fragile if LLM returns nested or escaped content.
- **Theory of Mind sections untouched** — if register doesn't fully recover, these are the next candidate. They also reference nearby characters (empathic sensing) but in a more somatic register than the relational/trust sections.
- **`DriveQualityEvalTest.java`** was accidentally committed with the evolved schema rendering change (was untracked from previous session). Review for appropriateness.

## Immediate Next Step

**Clear consolidation cache and run eval.**

1. Delete consolidation cache: `rm ~/claude/casehub/consolidation-cache/consolidation-cache-generic.json`
2. Start server with GENERIC profile, let consolidation rebuild (will now generate per-role descriptions)
3. Run 30-event scenario, save transcript
4. Classify with `classify_emotions.py`, compare register against Era 4 baseline (3.74)
5. If register recovers → 300-tick validation run
6. If register doesn't recover → investigate Theory of Mind sections as next candidate

**Install neocortex modules first** — the neocortex commits are on `main` but need `mvn install` before the examples app can see them. Run:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml install -Dmaven.test.skip=true -s examples/.mvn/slot-settings.xml
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml install -Dmaven.test.skip=true -s examples/.mvn/slot-settings.xml
```

## Cross-Module

- **neocortex main** — 3 commits for #175 (schema-aware descriptions, target resolution, targeted rendering). Not pushed to origin yet. Push before closing examples branch.
- **examples wip/era5-relational-eval** — 2 commits for #175 (evolved schema rendering, wiring). On top of era5 investigation commits.

## Era 5 Eval Results (from previous session — baseline for comparison)

| Run | Events | Quality | Register | Diff |
|-----|--------|---------|----------|------|
| Era 4 baseline | 300 | **3.92** | **3.74** | **3.74** |
| Era 5 + YAML briefing fallback | 30 | 3.84 | 3.27 | 3.70 |
| Era 5 label+degree only | 30 | 3.88 | 3.34 | 3.61 |
| Era 5 full pipeline (emerged descriptions) | 105 | 3.84 | 3.13 | 3.57 |

**Target:** Register should recover toward 3.74 with targeted somatic drives. Quality should hold or improve.

## Commits This Session

| Repo | Branch | Key commits |
|------|--------|-------------|
| neocortex | main | schema-aware DriveDescriptionGeneration (#175), TargetedDrive + resolveRelationalTargets (#175), targeted renderInternalState (#175) |
| examples | wip/era5-relational-eval | evolved renderRelationalSchemas (#175), wire 5-arg renderInternalState (#175) |
| wsp-casehub | main | plan, journal, handover |

## Key Files

| What | Where |
|------|-------|
| DriveDescriptionGenerationPhase | `neocortex/mindmap-intelligence/src/.../consolidation/DriveDescriptionGenerationPhase.java` |
| TargetedDrive | `neocortex/cognition/src/.../observation/TargetedDrive.java` |
| CognitionObservationBuilder | `neocortex/cognition/src/.../observation/CognitionObservationBuilder.java` |
| CharacterCognition | `examples/wacky-manor/src/.../agent/CharacterCognition.java` |
| Implementation plan | `wsp-casehub/plans/2026-10-29-targeted-somatic-drives.md` |
| Design journal | `wsp-casehub/JOURNAL.md` |
| Era 5 research | `neocortex/docs/research/era5-relational-cognition-research.md` |
| Consolidation cache | `~/claude/casehub/consolidation-cache/consolidation-cache-generic.json` |
| Eval data | `examples/wacky-manor/docs/eval/` |

## Constraints

- **Delete consolidation cache before eval.** The per-role descriptions only generate when descriptions don't exist yet (idempotent). Old cache has target-agnostic descriptions.
- **Install neocortex before running examples.** Changes are on neocortex main, need `mvn install` for examples to pick up.
- **Do not remove cognitive sections.** Relational schemas, attachment, trust, theory of mind — all stay. These represent abilities, not noise.
