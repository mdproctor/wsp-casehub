# HANDOFF — casehub-examples — 2026-10-13

## START HERE

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Layer 4 data tuning attempted → architectural pivot to `serves` annotations
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

**Read the research document before doing anything else:**
`wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`

This is the complete knowledge base for the next iteration. It covers:
- All literature findings (Conway SMS, Singer SDMs, Klinger, CAPS, LLM persona research) with specific citations and implementation mapping
- Full memory audit results (26 memories, per-memory findings)
- All eval data (L4a mean 3.79, L4b mean 3.82, PW mean 3.70, B0 mean 3.96)
- The `serves` annotation design (syntax, vocabulary, degree scale, parser behavior)
- The testing pyramid (Levels 0-5, what exists, what's needed)
- Implementation plan (Phase 0a-0e)
- Every relevant file path in both repos

**Design specs:**
- `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-09-drive-integration-architecture-design.md` — drive integration architecture (D1-D8)
- `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-concern-architecture-design.md` — formation memory concern architecture (D9-D11)
- `wsp-casehub/specs/issue-115-drive-expression-gap/decisions.md` — all 11 decisions

## What Needs to Happen Next

### 1. Implement the `serves` annotation system (Phase 0a-0e)

Full detail in the research document §7. In summary:

**Phase 0a:** Add `serves` + `ConcernAnnotation` to `FormationMemory` record in `SocialConfig.java`, update `ManorSocialConfigLoader` to parse, build annotation parser.

**Phase 0b:** Annotate all 26 formation memories in `social-config-generic.yaml` with `{{serves: drive|degree: text}}`. Use the audit results (research doc §3.2) for drive assignments.

**Phase 0c:** Fix the 11 pre-resolved conflicts, 6 action primings, 5 value declarations, 1 contradiction per audit findings.

**Phase 0d:** Build Level 2-4 micro tests for 5 collapsed drives (HC gloating, DD gloating, PP gallantry, Pen social-harmony, Pen adventure). `DriveExpressionEvalTest` has PP proving-worth as the pattern.

**Phase 0e:** Run full scenario (Level 5) to establish post-audit baseline. Compare to L4b (mean 3.82).

### 2. Close #117

Design implemented (commits `169b093`, `e30fe7a`). Close with comment referencing those commits.

### 3. Independent tasks

| Issue | What |
|-------|------|
| #116 | CognitivePipelineBuilder extraction |
| #114 | DD missing tendencies |

## Architecture Summary

| Layer | What | Status |
|-------|------|--------|
| 1 — Wiring | Connect dead ends in pipeline | **DONE** |
| 2 — Inference | CAPS labels, expression styles, appraisal foregrounding, Klinger salience | **DONE** |
| 3 — Data | Somatic drives, enriched memories, AMPD facets, disposition axes | **DONE** |
| 4 — Data tuning | Somatic refinement, formation memory enrichment | **ATTEMPTED → PIVOTED** |
| 0 — `serves` annotations | Annotate memories, audit, build micro tests, establish baseline | **NEXT** |
| 1-5 (drive integration) | Drive→CAPS, Drive→Appraisal, Drive→Memory salience, rendering cleanup | **BLOCKED on Phase 0** |

## Key Constraint: Emergence Not Scripting

**Strengthened from this session's findings:**

Formation memories are the strongest LLM behavioral signal (ACL 2025: demonstrations
override instructions). Every emotional evaluation in a memory overrides the personality
model. Do NOT:
- Pre-resolve drive conflicts ("not X, it was Y")
- Describe actions the character does (action priming)
- Declare what things mean (value declarations)
- State emotions that contradict what the personality model would derive

DO: describe situations. Let the personality model (drives, facets, CAPS network)
derive the emotional response. Use `serves` annotations to document which drive
a memory calibrates — for testing, not for the LLM.

The research document §3 has the full audit results per memory.

## Baseline Transcripts — DO NOT DELETE

| Transcript | Events | What it tests | Profile |
|---|---|---|---|
| `old-briefings-20261002/transcript.json` | 327 | Verbose briefings (pre-rewrite) | BASELINE |
| `new-briefings-20261002/transcript.json` | 333 | Directive-minimal (post-rewrite) | BASELINE |
| `generic-briefings-20261002/transcript.json` | 306 | Generic + prescribed catchphrases | GENERIC |
| `generic-colloquial-20261002/transcript.json` | 310 | Generic + style-directive only | GENERIC |
| `full-run-2026-10-08.json` | — | Full run baseline | — |
| `pipeline-hardening-run-2026-10-08/transcript.json` | — | Pipeline hardening run | — |
| `layer-123-run-2026-10-09/transcript.json` | 326 | Layer 1+2+3 baseline | GENERIC |
| `pipeline-wiring-run-2026-10-13/transcript.json` | 997 | Full pipeline wiring | GENERIC |
| `layer4-refinement-run-2026-10-13/transcript.json` | 349 | L4a somatic + memory refinements | GENERIC |
| `layer4-gallantry-fix-2026-10-13/transcript.json` | 300 | L4b gallantry somatic fix | GENERIC |

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```

## Blog Entries

- `wsp-casehub/blog/2026-10-13-mdp01-when-your-characters-stop.md` — Part 1: data tuning trap
- `wsp-casehub/blog/2026-10-13-mdp02-from-authoring-to-architecture.md` — Part 2: literature → architecture
