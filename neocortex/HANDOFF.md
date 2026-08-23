# Handoff — 2026-08-20 (issue-202-retrain-strategy-classifier)

## Last Session

Replaced BatchNorm with LayerNorm. Fixed pipeline label mismatch bug (phantom output classes from consolidation). Built dual-encoder architecture with separate Conv stacks for player/opponent features, learned sigmoid gating, feature projection (119→64), residual 4-head attention. Added label smoothing (0.1) and calibrated modality dropout (20%→40%). Per-source diagnostic revealed MSC (player-only) at 42.3% is the structural bottleneck — player features are an indirect proxy for the opponent's strategy. Results: vs_terran 52.8% (from 51.4%), vs_zerg 70.2% (from 66.5%), vs_protoss 74.0% (from 69.9%).

## Immediate Next Step

Batch 5 in the .plan: implement hierarchical classification head for vs_terran. If that doesn't break 65%, the bottleneck is temporal window length (strategy divergence happens at minute 5-7 but data only covers ~3-5 minutes). Extending windows requires re-extracting from raw replay archives with `max_windows > 10` in HyperParams — a significant effort (batch 4 tasks 9-10 are blocked on this).

## Cross-Module

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## Notes

- Pipeline now saves `classes.json` per matchup in `data/combined/` — run_pipeline reads it automatically
- `--min-samples 250` needed for normalize to trigger AIR_SUPERIORITY→MACRO_ECONOMY consolidation (243 samples)
- vs_protoss has 3 near-zero classes (DT_RUSH 6.8%, BLINK_STALKER 4.5%, AIR_SUPERIORITY 8.9%) — may need similar consolidation
- Garden push has unpushed commits — resolve on next garden maintenance
