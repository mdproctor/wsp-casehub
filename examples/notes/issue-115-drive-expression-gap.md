# Issue #115 — Drive Expression Gap

**Branch:** `issue-115-drive-expression-gap`
**Research doc:** `specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`
**Decisions:** `specs/issue-115-drive-expression-gap/decisions.md` (D1-D11)
**Blog:** 3-part series in `blog/` — "When Your AI Characters Stop Being Themselves", "From Authoring to Architecture", "When Drives Compete"

## Current Position

All 7 derivation chain breaks FIXED. Drive competition detection implemented and validated.

| Break | What | Status |
|-------|------|--------|
| #1 | INPUT nodes had no vocabulary | FIXED (Layer A) |
| #2 | Vocabulary was flat (no weights) | FIXED (Layer A) |
| #3 | 17 INPUT nodes had zero connections | FIXED (Layer A) |
| #4 | Placeholder catalogueEntryId | FIXED (Layer B) |
| #5 | situationTypes generic | FIXED (Layer B) |
| #6 | No substrate rendering | FIXED (Layer B) |
| #7 | Drive→CAPS activation not implemented | FIXED (Layer C) |

## Layer Status

| Layer | What | Status | Key Commits |
|-------|------|--------|-------------|
| A | Weighted situation vocabulary | DONE | neocortex `989647c6`, `bb4147dd` |
| B | Memory calibration | DONE | examples `f3c9673` (2a), `23eb356` (2b) |
| C | Drive amplification | DONE | neocortex `11e64632` |
| — | Drive competition detection | DONE | neocortex `ea9d6c91` (DriveCompetitionDetector), `87214417` (preamble) |
| — | Wiring unit tests (37) | DONE | examples `764c65c` |

## Drive Competition — The Key Discovery

Drives compete when they share CAPS activation nodes (§10.10). Overlapping pathways produce similar felt experience → LLM conflates them → defaults to the most concrete drive.

**What was tried (4 iterations):** substrate tuning (ceiling at 3/5), per-character meta-text (5/5 but prescriptive), CAPS-overlap detection with conditional instruction (5/5 but static table), preamble-level universal instruction (4-5/5, correct architecture).

**Current state:** Universal meta-instruction in `CognitivePreambleGenerator` (neocortex cognition module): "Each drive is a distinct felt state. When multiple drives activate in the same moment, attend to which quality of feeling is at the front." Always present when drives are enabled. No detection gate, no per-character config.

**What's not right yet:** The meta-instruction is generic ("warmth, tension, or urgency") — operates at S3 (Semerari MAS-A). Should be DERIVED from the character's actual CAPS pathways and formation memories during consolidation to reach S6 (narrative-grounded). The D1 static mapping table (`DriveCompetitionDetector`) should be replaced by inference-based detection using the vocabulary classifier. See §10.10.6-7 for literature review and design.

**Literature review DONE (§10.10.6):** Barrett (2017), Kashdan et al. (2015), Semerari et al. (2003). Convergent finding: differentiation requires grounded conceptual knowledge (S6), not generic metacognitive instruction (S3). Key revision: classify formation memories (not drive descriptions) as primary signal. Design in §10.10.7, decision D12.

## Micro Test Results (Layer B Eval — 2026-10-14)

| Test | P0 | Layer B+C | With Preamble Meta |
|------|----|-----------|--------------------|
| HC gloating | 5 | 5 | 5 |
| DD gloating | 5 | 5 | 5 |
| PP gallantry (door) | 3 | 2→3 | 4-5 |
| PP gallantry (differentiation) | 3 | 4 | 5 |
| Pen harmony | 5 | 5 | 5 |
| Pen adventure | 5 | 5 | 5 |

Results saved: `wacky-manor/docs/eval/layer-b-eval-2026-10-14/`

## What's Next

1. ~~**Literature review**~~ — **DONE** (§10.10.6). Three traditions converge: differentiation requires S6-level grounded conceptual knowledge (formation-memory-traced), not S3-level generic instruction. Key revision: classify formation memories (not drive descriptions) as primary signal.
2. **Implement inference-based drive differentiation** (§10.10.7, D12) — `DriveProfileDerivationPhase` as new `ConsolidationPhase`. Two layers: inferred drive CAPS profiles (replaces static `DRIVE_CAPS_MAPPINGS`), grounded differentiation text (replaces generic `DIFFERENTIATION_INSTRUCTION`). Design complete — ready for implementation.
3. **Full scenario run** (step 3b) — AFTER inference system is built. No point validating against a configuration we're about to change. One run against the final form.
4. **Blog** — Part 3 written and committed. Dedicated article on drive competition once inference system is validated.

## What NOT to Re-Derive

- D1-D11 decisions (made, in `decisions.md`)
- Serves annotation system (working, tested)
- Vocabulary weights (first approximations, empirically tunable)
- Memory-to-catalogue mappings (§12.5 is authoritative)
- Memory-to-catalogue methodology (§12.4 — three tests, OUTPUT-vs-MEDIATING rule)
- Three-layer approach (situation encoding, memory calibration, drive amplification)
- Drive competition mechanism (§10.10 — CAPS overlap detection)
- Literature (all cited in research doc)

## Eval Baselines

| Metric | Value |
|--------|-------|
| B0 (BASELINE0) mean | 3.96 |
| P0 (Phase 0 post-audit) mean | 3.72 |
| L4b (Layer 4 somatic fix) mean | 3.82 |
| Target | >= B0 (3.96) via emergent mechanisms |
