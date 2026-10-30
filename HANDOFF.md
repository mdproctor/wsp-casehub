# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-30
**Branch:** wip/era5-relational-eval (examples), main (neocortex)
**Era:** 5 — relational cognition, targeted somatic drives
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## This Session

Deep first-principles investigation of register gap (3.46 vs 3.74 Era 4 baseline). Created golden prose corpus as pipeline regression target.

### Key Findings

1. **Section structure is identical between Era 4 and Era 5.** All 15 cognitive section slots existed in Era 4. The problem is not new sections — it's what goes INTO them.

2. **Relational schemas NOT seeded into MindMap.** `seedPeople()` creates overlays with only PAD values. No role/trust/intimacy/utility. Consequence: P20 has nothing to evolve (0ms), P22 finds 0 schemas, per-role descriptions never generated.

3. **"What Pulls at You" grows from 9→25 parts** after consolidation (4 drives + 4 self-standards + 5 attractors + 6 differentiation + 6 priority signatures). Register degrades as content volume increases — Era 4 also shows this degradation over time.

4. **Era 5 drops action readiness channel to zero.** Era 4 had action readiness in every aside (1.9 markers avg). Era 5 has zero. The analytical relational sections provide ready-made labels that short-circuit action readiness.

5. **Barrett 4-channel selection is per drive×role×situation**, not exhaustive. Each drive has a primary and secondary channel. The target role modulates which channels foreground.

### Golden Prose Corpus

51 events, 5 characters, 10 situations each. **Q=4.35, R=4.37, D=4.12** — beats Era 4 baseline on all dimensions.

Location: `wacky-manor/docs/eval/golden-prose-20261030/`
Regression gate: `python3 verify_golden.py [--verbose]`

**Per-drive gaps still open (sub-4.0):**

| Character | Drive | Q | R | D |
|-----------|-------|:-:|:-:|:-:|
| dick-dastardly | scheming | 3.62 | 3.88 | 3.38 |
| hooded-claw | scheming | 4.44 | 3.89 | 4.44 |
| hooded-claw | self-preservation | 3.80 | 4.20 | 4.00 |
| hooded-claw | gloating | 4.50 | 4.50 | 3.75 |

Over-trigger reduction was tried and FAILED for both villains — their drives are identity-level. Needs a different approach: adding genuinely non-scheming events rather than subtracting scheming from existing ones.

### Methodology Established

Every cognitive extension must follow: **describe contribution to rendered output → test it scores above threshold → then implement the backend**. Not the reverse.

### Architecture Agreed

**Drive-first rendering structure:**
```
DRIVE (intensity) → Target Name
[Barrett channel-selected description — 1-2 channels, not all 4]
```

**Channel selection is mechanical** — lookup table:
- Drive type → primary/secondary channel
- Target role → channel boost
- No LLM needed for selection

**Relational information integrates INTO drives**, not alongside:
- Trust → shapes intensity within channels
- ToM → integrates into drive expression
- Social awareness → becomes felt, not classified
- Relational schema → provides the role that selects channels

**Per-tick pipeline is almost entirely mechanical.** Only LLM call is situation classification. Appraisal, filtering, targeting, rendering — all deterministic.

### Pipeline Principles Discovered

1. **Body-location differentiation** — each drive needs unique body location
2. **Trigger-rate modulation** — fewer triggers at higher quality beats ubiquitous low quality (but doesn't apply to identity-level drives)
3. **Temporal discovery ordering** — "first...then" beats co-present description
4. **Drives peak when owning events** — competing drives dilute savoring
5. **Channel selection by situation** — same drive, different channels in different contexts

## Immediate Next Steps

1. **Get golden prose to all-4.0+ per drive** — villains need different approach (new events, not subtraction). Fresh session recommended.
2. **Seed relational schemas into MindMap** — write role/trust/intimacy/utility into overlay properties in `seedPeople()`
3. **Restructure renderCognitiveSections** — consolidate 5 fragmented relational sections into drive-first integrated blocks
4. **Update DriveDescriptionGenerationPhase** — include channel emphasis hints per drive×role
5. **Run eval** — compare pipeline output against golden corpus scores

## Commits This Session

| Repo | Branch | Key commits |
|------|--------|-------------|
| examples | wip/era5-relational-eval | era5-targeted eval data, ideal prose baseline, golden prose corpus (3 iterations), methodology |
| wsp-casehub | wip/era5-relational-eval | handover updates |
| neocortex | main | (no changes this session) |

## Key Files

| What | Where |
|------|-------|
| Golden prose corpus | `examples/wacky-manor/docs/eval/golden-prose-20261030/` |
| Regression gate | `golden-prose-20261030/verify_golden.py` |
| Methodology | `golden-prose-20261030/methodology.md` |
| Channel selection tables | This handover (Architecture Agreed section) |
| Pipeline principles | This handover (Pipeline Principles section) |
| Format evolution issue | casehubio/examples#176 (Traitors/Big Brother) |
| Era 5 research | `neocortex/docs/research/era5-relational-cognition-research.md` |

## Constraints

- **Do NOT remove cognitive sections.** They represent abilities. Fix the rendering, not the cognition.
- **Do NOT accept regressions.** Golden means every dimension improves or holds.
- **Describe → test → implement.** Every cognitive extension proves its contribution to rendered output before backend work.
- **Delete consolidation cache before eval runs.** Old cache has target-agnostic descriptions.
- **Install neocortex before running examples.** Changes on neocortex main need `mvn install`.
