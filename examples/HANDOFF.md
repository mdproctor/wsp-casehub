# HANDOFF — casehub-examples — 2026-10-13

## START HERE

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Phase 0 complete. Root cause found. Theory validated by micro tests. Implementation plan for Phase 1 ready.
**Plan:** `.plan` has #115 active (covers=115,118,119)

### Read Order for Next Session

1. **This file** — orientation and "do not re-derive" list
2. **Research doc §0 (Current State)** — 30-line summary of where we are
3. **Research doc §10 (Situation Encoding Investigation)** — root cause and three-layer approach
4. **Research doc §12 (Phase 1 Implementation Plan)** — concrete steps

**Research doc:** `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`

Do NOT read the full 800-line research doc upfront. §0 tells you what's settled. §10 and §12 tell you what to do. Read §1-§9 only if you need supporting evidence for a specific question.

### Do Not Re-Derive

These are done. Do not re-investigate, re-brainstorm, or re-litigate:

| What | Status | Where |
|---|---|---|
| D1-D11 design decisions | Made | `decisions.md` |
| `serves` annotation system | Built and tested | Phase 0a-0b (commits `85e6d3a`, `dcfdf2c`) |
| Formation memory audit | Complete (23 fixes) | Phase 0c (commit `9bc9dec`) |
| Micro test baselines | Captured | §8.1, §8.1b — 11 tests, all scored |
| Full scenario baselines | Two runs averaged | §8.2 — mean 3.72 |
| Root cause: empty INPUT keywords | Found and documented | §10.1 |
| Literature: situation encoding in CAPS | Researched | §10.2 (Read & Miller, Shoda/Mischel, Sacchi/Dan-Glauser) |
| Three-layer derivation approach | Defined | §10.7 |
| Intensity principle | Established | §10.8 |
| Emergent gloating derivation | Validated by micro tests | §10.9, §8.1b |
| Redundancy principle | "Include all layers; prove redundancy empirically later" | §8.1b |

### The Root Cause (Summary)

The CAPS network has **zero content-based situation activations**. All 28 INPUT nodes have empty keyword/vocabulary lists. The `RuleBasedSituationClassifier` skips nodes with empty vocabularies. The network settles on personality weights alone — it can distinguish personality types but NOT situations.

Micro tests prove the LLM expresses all drives at 5/5 when given a direct trigger situation. In full scenarios, events go through the pipeline and lose situational specificity. The personality substrate is already expressive enough — the pipeline just can't see what's happening.

### The Fix (Three Layers)

**Layer A — Situation Vocabulary:** Add weighted, construct-derived vocabulary to the 28 CAPS INPUT nodes. Terms derived from each node's source literature (McClelland for achievement, Bowlby for attachment, Gray RST for threat, etc.) at graded activation weights. Universal across characters — personality differences emerge from propagation, not encoding.

**Layer B — Memory Calibration:** Connect formation memories to actual catalogue entries (replace placeholder `catalogueEntryId`). Catalogue entries calibrate mediating nodes (`self_worth`, `self_efficacy`, `other_reliability`). Include explicit mediating substrate rendering so the LLM sees the psychological depth (insecurity, validation need) alongside drives.

**Layer C — Drive Amplification:** Implement D1 — drive intensities inject persistent activations into CAPS nodes. Drives co-activate with situation signals, amplifying drive-relevant events for characters with relevant drives.

All three layers needed. Do not assume any is redundant without empirical proof.

### Key Constraint

All mechanisms must be **literature-rooted** (CAPS theory, Mischel's if-then profiles), **individual-specific** (personality determines activation, not universal rules), **not hard-wired** (no "rival fails → gloating" rules), and **LLM-bias-aware** (extreme substrate overcomes LLM sanitization tendency). Include redundant signals; prove redundancy empirically — never assume it.

## Phase 0 Summary (Complete)

| Phase | Commit | What |
|---|---|---|
| 0a | `85e6d3a` | `ConcernAnnotation` record, `FormationMemoryParser`, loader + renderer |
| 0b | `dcfdf2c` | Annotated all 26 formation memories with `serves` annotations |
| 0c | `9bc9dec` | Fixed 11 pre-resolutions, 6 action primings, 5 value declarations, 1 contradiction |
| 0d | `a83cfba` + `02d008a` | 6 drive expression tests + 5 substrate tests, baselines captured |
| 0e | `bab91f7` + `58de5db` | Two full scenario runs, classified, two-run averages established |
| fix | `a2f5e9a` | Restored Pen age-7 and DD age-6 felt intensity |

**Phase 0 eval (two-run average, mean 3.72):** PP gallantry +0.48 vs L4b (recovering). HC/DD gloating flat (+0.06/+0.02, confirmed pipeline gap). Pen social-harmony -0.05 (stable). Pen adventure +0.13.

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
| `phase0-audit-baseline-2026-10-13/transcript.json` | 294 | Phase 0 run 1 (post-audit) | GENERIC |
| `phase0-fix-run-2026-10-13/transcript.json` | 241 | Phase 0 run 2 (felt-intensity fix) | GENERIC |

## Build Notes

```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```
