# Issue #115 — Drive Expression Gap

**Branch:** `issue-115-drive-expression-gap`
**Research doc:** `specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`
**Decisions:** `specs/issue-115-drive-expression-gap/decisions.md` (D1-D11)

## Current Position

**Layer B complete (2026-10-14).** All 7 derivation chain breaks mapped; 4 fixed, 1 remaining.

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

| Layer | What | Status | Commits |
|-------|------|--------|---------|
| A | Weighted situation vocabulary | DONE | neocortex `989647c6`, `bb4147dd` |
| B | Memory calibration | DONE | examples `f3c9673` (2a), `23eb356` (2b) |
| C | Drive amplification | DONE | neocortex `11e64632` — D1 mappings in `BehavioralSynthesisPhase` |

## Layer C — DONE

D1 implemented: `BehavioralSynthesisPhase.mergeDriveIntensities()` now uses explicit mappings (14 drive types → CAPS nodes) instead of text classification. Unmapped types fall back to vocabulary classifier.

## All Three Layers Complete — What's Next

All 7 derivation chain breaks are fixed. Remaining steps:
1. **Install neocortex** with Layers A+C changes, rebuild wacky-manor
2. **Run full scenario** (step 3b) — compare against P0 baseline (mean 3.72)
3. **Classify with `classify_emotions.py`** — measure per-drive scores, compare to B0 (mean 3.96)
4. If gaps remain, empirically tune vocabulary weights (D4a) or adjust substrate descriptions

## Key Methodology — Repeatable

**Memory-to-catalogue mapping (§12.4):** three tests (directional correctness, cumulative trajectory, no contradictions), OUTPUT-vs-MEDIATING rule. Always prefer entries that calibrate MEDIATING nodes. Applied to all 26 memories across 5 characters.

**Substrate descriptions:** felt-experience language, no behavioral prescriptions. Derived from the mapping analysis, not from intuition. Connected to formation memories as "What Shaped You" section.

## What NOT to Re-Derive

- D1-D11 decisions (made, in `decisions.md`)
- Serves annotation system (working, tested)
- Vocabulary weights (first approximations, empirically tunable)
- Memory-to-catalogue mappings (§12.5 is authoritative)
- Three-layer approach (situation encoding, memory calibration, drive amplification)
- Literature (all cited in research doc — CAPS, Read & Miller, Conway SMS, etc.)

## Eval Baselines

| Metric | Value |
|--------|-------|
| B0 (BASELINE0) mean | 3.96 |
| P0 (Phase 0 post-audit) mean | 3.72 |
| L4b (Layer 4 somatic fix) mean | 3.82 |
| Target | >= B0 (3.96) via emergent mechanisms, not prescriptive text |

## Blocked / Dependent

- Layer C blocks step 3b (full scenario with all three layers)
- #117 close is independent (design already implemented)
- #116, #114 are independent
