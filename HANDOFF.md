# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Inference-based drive differentiation implemented. Pivot to inferred somatic rendering (#120) as next step.
**Plan:** `.plan` has #115 active (covers=115,118,119), #117 + #120 queued

## START HERE

**Operational note (progress, direction, what not to re-derive):**
`wsp-casehub/examples/notes/issue-115-drive-expression-gap.md`

**Research document (deep technical reference, literature, methodology):**
`wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`

**How to read the research doc:**
- **§0** — current state, what's done, what's settled (do not re-derive), what's next
- **§10.10.6** — literature review: Barrett (2017), Kashdan (2015), Semerari (2003) — emotion differentiation
- **§10.10.7** — design: inference-based drive differentiation (D12)
- **§10.10.8** — design: inferred somatic rendering (#120, D13) — the pivot
- **§12.4** — memory-to-catalogue mapping methodology (repeatable)
- **§12.5** — authoritative mapping table (26 memories, 5 characters)

The note gives you operational context. The research doc gives you depth. Read the note first.

## What Was Done This Session (2026-10-14, session 2)

### Literature Review (§10.10.6)
- Barrett (2017), Kashdan et al. (2015), Semerari et al. (2003) reviewed
- Convergent finding: differentiation requires S6-level grounded conceptual knowledge (formation-memory-traced), not S3-level generic instruction
- Key revision: classify formation memories (not drive descriptions) as primary signal
- Decision D12 recorded

### DriveProfileDerivationPhase (§10.10.7, D12)
- New `ConsolidationPhase` in neocortex mindmap-intelligence (priority 20)
- Classifies formation memories through vocabulary classifier → per-drive CAPS profiles
- Detects competition via MEDIATING pathway overlap
- Derives S6-grounded differentiation text from unique formation memories
- Universal template: memory excerpts + "Both are real. They're not the same." — no mechanism bias
- 8 unit tests, 0.07s
- Render wiring: `CharacterCognition.renderInternalState()` reads `DriveCompetitionInferred` artifacts

### Micro Test Campaign (13+ LLM eval runs)
- Pre-inference baseline: PP gallantry door 2-3/5, dinner 5/5, all other drives 5/5
- **Discovery 1: Proximity** — meta-instruction must be adjacent to drives in observation section. Commit `daf6272` moved it to preamble, losing the effect.
- **Discovery 2: S6 > S3** — grounded text (memory-traced, 4/5) outperforms generic instruction (2-3/5)
- **Discovery 3: Template bottleneck** — hand-authored somatic descriptions create echo-scripting and concreteness asymmetry. Protection's "a loose hinge" primes fixing behavior in hinge scenarios.
- Temporal and gestural S6 variants both scored 4/5 — adopted universal template without mechanism bias
- Results saved: `wacky-manor/docs/eval/pre-inference-2026-10-14/`

### Pivot to Inferred Somatic Rendering (#120, §10.10.8, D13)
Hand-authored drive descriptions in social-config.yaml are the bottleneck, not the differentiation text. New issue #120 tracks: LLM-generated somatic descriptions during consolidation + situation-specific priming at render time (Conway SMS + Barrett constructed emotion). No template text — the LLM constructs felt states from formation memories + situation context.

### Cross-Repo: neocortex `issue-115-weighted-situation-vocabulary`

| Commit | What |
|--------|------|
| `caae5ba9` | DriveProfileDerivationPhase — memory-traced drive differentiation |
| `10e1f88a` | Universal differentiation template — no mechanism bias |

### Examples `issue-115-drive-expression-gap`

| Commit | What |
|--------|------|
| `944b612` | Pre-inference baseline eval results |
| `4e8de22` | S3 vs S6 adjacent-placement tests |
| `a9cd8b2` | S6 sub-header format validated |
| `ca2bc4e` | Wire renderInternalState to read DriveCompetitionInferred artifacts |
| `2771526` | S6 temporal + gestural variants, buildObservationWithDriveMeta helper |

### Issue #115 comment
Literature review findings posted to GitHub issue.

### Issue #120 created
Inferred somatic rendering — LLM-generated drive descriptions from formation memories.

**Note:** `caps-llm-classifier` test has pre-existing compilation error from Layer A vocabulary change (keywords→vocabulary). Needs fix before merge.

## What Needs to Happen Next

### 1. Inferred Somatic Rendering (#120, §10.10.8, D13)
Replace hand-authored drive descriptions with LLM-generated, situation-primed somatic text. Two phases: (1) LLM generates base descriptions per drive per character during consolidation (precedent: BeliefRevisionPhase), (2) situation priming selects relevant formation memories at render time. Start by reading `BeliefRevisionPhase` to understand LLM access pattern during consolidation.

### 2. Full Scenario Validation (step 3b)
AFTER #120. Install neocortex, rebuild wacky-manor, run GENERIC scenario (~300 events), classify with `classify_emotions.py`. Compare to P0 (3.72) and B0 (3.96).

### 3. Blog
Parts 1-3 written. Next blog must cover: literature review (Barrett/Kashdan/Semerari), S3→S6 progression, proximity discovery, pivot from template to inferred somatic rendering, constructed emotion theory applied to cognitive agents.

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
| — Drive competition | Inference-based detection + S6 differentiation text | **DONE** (DriveProfileDerivationPhase) |
| — Somatic rendering | LLM-generated, situation-primed drive descriptions | **NEXT** (#120) |

### The Derivation Chain (all breaks fixed)

```
Formation Memories (serves: recognition|strong)
  → [catalogue entry] calibrates MEDIATING nodes            ← Break #4: FIXED (Layer B)
  → [situation occurs] INPUT nodes fire via vocabulary       ← Break #1: FIXED (Layer A)
  → [propagation] INPUT→MEDIATING connections exist          ← Break #3: FIXED (Layer A)
  → [drive activation] persistent drive bias co-activates   ← Break #7: FIXED (Layer C)
  → [settling] OUTPUT attractors emerge
  → [rendering] LLM receives attractor + memory salience → behaviour
  → [differentiation] competing drives distinguished via S6 grounded text ← NEW
  → [somatic] drive felt-state inferred from memories + situation ← NEXT (#120)
```

## Eval Baselines

| Metric | Value | Source |
|---|---|---|
| B0 (BASELINE0 prescriptive) mean | 3.96 | `classify_emotions.py` on old-briefings run |
| P0 (Phase 0 post-audit) mean | 3.72 | Two-run average (§8.2) |
| L4b (Layer 4 somatic fix) mean | 3.82 | Single run |
| PP gallantry door (raw, no meta) | 2-3/5 | Pre-inference baseline (this session) |
| PP gallantry door (S6 grounded) | 4/5 | Adjacent placement, memory-traced |
| PP gallantry dinner (no fix) | 5/5 | Gallantry emerges when proving-worth can't activate |

## Build Notes

**Correct build order:**
```
platform → eidos → engine (-Dmaven.test.skip=true) → work/progress-api → qhorus → neocortex → blocks (exclude engine-adapter) → wacky-manor (-f wacky-manor/pom.xml)
```

**Neocortex must be on `issue-115-weighted-situation-vocabulary` branch** for all Layer A/C/preamble/DriveProfileDerivationPhase changes. Install with `mvn install -Dmaven.test.skip=true` before building wacky-manor.
