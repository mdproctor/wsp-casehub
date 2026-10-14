# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** All layers complete. Drive competition detection implemented. Full scenario validation run next.
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 queued

## START HERE

**Operational note (progress, direction, what not to re-derive):**
`wsp-casehub/examples/notes/issue-115-drive-expression-gap.md`

**Research document (deep technical reference, literature, methodology):**
`wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`

**How to read the research doc:**
- **§0** — current state, what's done, what's settled (do not re-derive), what's next
- **§10.10** — drive competition discovery, what was tried, literature, architecture vision
- **§12.4** — memory-to-catalogue mapping methodology (repeatable)
- **§12.5** — authoritative mapping table (26 memories, 5 characters)

The note gives you operational context. The research doc gives you depth. Read the note first.

## What Was Done This Session (2026-10-14)

### Layer B — Memory Calibration
- Mapped all 26 formation memories to catalogue entries (§12.5)
- Developed repeatable methodology: three tests, OUTPUT-vs-MEDIATING rule (§12.4)
- Added `catalogue` field to `FormationMemory`, enriched `situationTypes` with drive annotations
- Added "What Shaped You" psychological substrate section for all 5 characters
- 37 wiring unit tests (`LayerBCWiringTest`) — deterministic validation, 0.18s

### Layer C — Drive Amplification
- D1 drive→CAPS activation injection in `BehavioralSynthesisPhase`
- 14 drive types → specific CAPS nodes, text classification as fallback
- Neocortex commit: `11e64632`

### Drive Competition Detection (§10.10)
- Discovered: drives compete when they share CAPS activation nodes
- Built `DriveCompetitionDetector` in neocortex caps-api (commit `ea9d6c91`)
- Tried 4 approaches: substrate tuning → per-character meta → CAPS detection → preamble instruction
- Final form: universal meta-instruction in `CognitivePreambleGenerator` (commit `87214417`)
- PP gallantry: P0 baseline 3/5 → now 4-5/5

### Blog
- Part 3: "When Drives Compete" — covers implementation, discovery, literature, architecture vision
- Series markers with relative links added to all 3 parts

### Cross-Repo Changes (neocortex `issue-115-weighted-situation-vocabulary`)

| Commit | What |
|--------|------|
| `989647c6`, `bb4147dd` | Layer A: weighted vocabulary, 30 missing connections |
| `11e64632` | Layer C: D1 drive→CAPS injection |
| `ea9d6c91` | DriveCompetitionDetector in caps-api |
| `87214417` | Drive differentiation in CognitivePreambleGenerator |

**Note:** `caps-llm-classifier` test has pre-existing compilation error from Layer A vocabulary change (keywords→vocabulary). Needs fix before merge.

### Cross-Repo: DriveProfileDerivationPhase (neocortex `issue-115-weighted-situation-vocabulary`)

| Commit | What |
|--------|------|
| `caae5ba9` | DriveProfileDerivationPhase — memory-traced drive differentiation (§10.10.7, D12) |

New `ConsolidationPhase` (priority 20, after BehavioralSynthesisPhase at 19). Classifies formation memories through vocabulary classifier, infers per-drive CAPS profiles, detects competition via MEDIATING pathway overlap, derives S6-grounded differentiation text. 8 unit tests.

## What Needs to Happen Next

### 1. Implement Inference-Based Drive Differentiation (§10.10.7, D12)
`DriveProfileDerivationPhase` as new `ConsolidationPhase`. Two layers: (1) inferred drive CAPS profiles from formation memory classification (replaces static `DRIVE_CAPS_MAPPINGS`), (2) S6-grounded differentiation text from unique formation memories (replaces generic S3 `DIFFERENTIATION_INSTRUCTION`). Design complete in §10.10.7. Cold-start fallback to existing static table.

### ~~2. Literature Review~~ — DONE (§10.10.6)
Barrett (2017), Kashdan et al. (2015), Semerari et al. (2003) reviewed. Convergent finding: differentiation requires grounded conceptual knowledge populated by specific experiential referents (formation memories), not generic metacognitive instruction. Key revision: classify formation memories (not drive descriptions) as primary signal. Decision D12 recorded.

### 3. Full Scenario Validation (step 3b)
AFTER the inference system is built. No point running against a configuration we're about to change. Install neocortex, rebuild wacky-manor, run GENERIC scenario (~300 events), classify with `classify_emotions.py`. Compare to P0 (3.72) and B0 (3.96). One run against the final form.

### 4. Independent Tasks
| Issue | What |
|-------|------|
| #117 | Close — design implemented |
| #116 | CognitivePipelineBuilder extraction |
| #114 | DD missing tendencies |

## Architecture Summary

| Layer | What | Status |
|-------|------|--------|
| A — Situation vocabulary | Weighted vocabulary for CAPS INPUT nodes | **DONE** |
| B — Memory calibration | Formation memories → catalogue entries → mediating nodes | **DONE** |
| C — Drive amplification | D1 drive→CAPS activation injection | **DONE** |
| — Drive competition | Metacognitive monitoring for overlapping drives | **DONE** (preamble) |

### The Derivation Chain (all breaks fixed)

```
Formation Memories (serves: recognition|strong)
  → [catalogue entry] calibrates MEDIATING nodes            ← Break #4: FIXED (Layer B)
  → [situation occurs] INPUT nodes fire via vocabulary       ← Break #1: FIXED (Layer A)
  → [propagation] INPUT→MEDIATING connections exist          ← Break #3: FIXED (Layer A)
  → [drive activation] persistent drive bias co-activates   ← Break #7: FIXED (Layer C)
  → [settling] OUTPUT attractors emerge
  → [rendering] LLM receives attractor + memory salience → behaviour
```

## Eval Baselines

| Metric | Value | Source |
|---|---|---|
| B0 (BASELINE0 prescriptive) mean | 3.96 | `classify_emotions.py` on old-briefings run |
| P0 (Phase 0 post-audit) mean | 3.72 | Two-run average (§8.2) |
| L4b (Layer 4 somatic fix) mean | 3.82 | Single run |

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```

**Neocortex must be on `issue-115-weighted-situation-vocabulary` branch** for all Layer A/C/preamble changes. Install with `mvn install -Dmaven.test.skip=true` before building wacky-manor.
