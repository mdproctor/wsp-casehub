# HANDOFF — casehub-neocortex

## Last Session

Fixed 3 pipeline gaps blocking drive discrimination in full scenario runs (#584, #585, #586). All work on branch `issue-115-weighted-situation-vocabulary`.

### What was fixed

**#584 — DriveProfileDerivationPhase produces 0 profiles:**
- Root cause: early-return guard at line 76 when `SituationClassifier`/`CapsTopology` CDI `Instance<>` deps don't resolve during consolidation
- Fix: graceful degradation — profiles derive from memory excerpts alone (no CAPS input activation); competition detected via shared `catalogue-entry-id` overlap instead of shared mediating nodes
- Commit: `33235928`
- Tests: 10/10 green (8 existing + 2 new)

**#586 — Somatic discrimination text not generating:**
- Root cause: cascading failure from #584 — 0 competition artifacts meant `generateDiscriminationText()` had nothing to process
- Fix: verified via cross-phase pipeline test — Phase 1 (null CAPS, catalogue fallback) → Phase 2 (LLM generation) works end-to-end
- Commit: `9c7ddea6`

**#585 — Generation prompt somatic-only strips discriminating channels:**
- Root cause: SYSTEM_PROMPT and DISCRIMINATION_SYSTEM both restricted to "somatic/bodily sensations only"
- Fix: updated to Barrett (2017) 4-channel constructed emotion — somatic, attentional, temporal, action tendency
- User prompts also updated to remove "somatic only" restriction
- Commit: `9c7ddea6` (same commit as #586)
- Tests: 25/25 green across both phase test classes

### Key decisions

- **Catalogue-based competition fallback**: when CAPS topology is unavailable, drives compete if their formation memories share `catalogue-entry-id` values. This is semantically correct — catalogue entries represent experience-behaviour patterns, so shared entries mean overlapping experiential territory.
- **Multi-channel prompts**: each channel (somatic, attentional, temporal, action tendency) should contribute something that distinguishes THIS pull from any other. The "somatic only" constraint was the direct cause of the B0 → pipeline regression (3.96 → 3.55).

## Immediate Next Step

Run a full scenario with GENERIC profile to measure whether the pipeline regression is recovered. Compare against B0 baseline (3.96 mean). The cognitive-dump endpoint shows what each character sees — verify that generated descriptions contain channel-specific discriminating vocabulary.

## References

- Epic: casehubio/examples#122 (Batch 1)
- Branch: `issue-115-weighted-situation-vocabulary` (neocortex)
- Branch: `issue-115-drive-expression-gap` (examples)
