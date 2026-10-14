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

**What's not right yet:** The meta-instruction is generic ("warmth, tension, or urgency"). It should be DERIVED from the character's actual CAPS pathways and formation memories during consolidation. The D1 static mapping table (`DriveCompetitionDetector`) should be replaced by inference-based detection using the vocabulary classifier. See §10.10.3-10.10.4 for details.

**Literature to investigate:** Barrett (2017) *How Emotions Are Made* — emotion differentiation / granularity. Kashdan et al. (2015) "Unpacking Emotion Differentiation." Semerari et al. (2003) Metacognition Assessment Scale. These may provide the operationalisation for inference-based drive differentiation.

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

1. **Full scenario run** (step 3b) — install neocortex (Layers A+C+preamble), rebuild wacky-manor, run GENERIC scenario (~300 events), classify with `classify_emotions.py`, compare to P0 (3.72) and B0 (3.96)
2. **Inference-based drive differentiation** — during consolidation, classify drive descriptions through vocabulary classifier → detect CAPS overlap → trace pathways through formation memories → derive grounded differentiation text
3. **Literature review** — Barrett, Kashdan, Semerari on emotion differentiation / metacognitive monitoring
4. **Blog** — Part 3 written and committed. Consider dedicated article on drive competition if inference system works.

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
