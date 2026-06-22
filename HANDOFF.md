# HANDOFF — casehub

**Date:** 2026-06-22
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**parent#93 closed — Agent Mesh Primitives extracted to casehub-engine-api**

`CaseChannelLayout`, `NormativeChannelLayout`, `SimpleLayout`, `MeshParticipationStrategy`, and three standard participation strategy implementations moved from `claudony-casehub` to `io.casehub.api.spi.mesh` in `casehub-engine-api`. OpenClaw's duplicate `OpenClawNormativeLayout` deleted. Local protocol promoted to garden.

Changes across 4 repos (all on main):
- **engine** (#550): 11 new files, 446 tests. `strategyFor(String, UUID)` signature fixed (was nullable WorkerContext wrapper). null guards on `named(null)`, contract tests cover both layouts, tautological enum tests removed (#554).
- **claudony** (#159): 10 files deleted, `selectStrategy()` removed, `MeshParticipationStrategy.named()` replaces it. 143 tests.
- **openclaw** (#38): `OpenClawNormativeLayout` deleted, providers refactored. Reactive `openOrCreate(UUID, ChannelSpec)`. Unknown purposes throw IAE. 150 tests.
- **parent**: PLATFORM.md (2 new rows), CHANNELS.md (MeshParticipationStrategy formalised), casehub-engine.md deep-dive updated.
- **garden**: `spi-case-id-parameter.md` rows updated, `normative-channel-layout-single-source.md` created (platform scope).

ARC42STORIES updated in all three workspaces (engine/claudony/openclaw).

## Immediate Next Step

**parent#288** — Add casehub-worker and casehub-platform-governance artifacts to the BOM + update `docs/PLATFORM.md`. Unblocks `engine#543` (Worker migration, 60+ files).

## Cross-Module: Worker Foundation Extraction

**Tracking issue:** casehub-desiredstate#40

| Step | Repo | Issue | Status |
|------|------|-------|--------|
| 0a | casehub-platform | platform#104 | Done (branch needs merge) |
| 0b | casehub-worker | — | Done (repo created, published) |
| 1 | **casehub-parent** | **parent#288** | **Next — BOM + PLATFORM.md** |
| 2 | casehub-engine | engine#543 | Blocked on parent#288 |
| 3 | casehub-desiredstate | desiredstate#41 | Blocked on engine#543 |
| 4 | claudony | claudony#157 | Blocked on engine#543 |
| 5 | casehub-workers | workers#14 | Blocked on engine#543 |
| 6 | casehub-openclaw | openclaw#37 | Blocked on engine#543 |
| 7 | casehub-ops | ops#8 | Blocked on engine#543 |

## What's Left

- platform#107 — remove cloudevents version pins from platform root pom (now managed by parent BOM)
- platform#104 — governance types branch needs merge + publish
- #210 — casehub-rag-api cross-dep rows — blocked until engine/eidos wire the dep

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #288 | casehub-worker + governance BOM + PLATFORM.md | S | Low | Immediate — unblocks engine#543 |
| #277 | CloudEvent infrastructure rollout epic | L | Med | Adapters done; track consumer adoption |
| #293 | Formalise channel taxonomy (CHANNELS.md) | S | Low | Partially done this session |
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon design |
| engine#556 | PerCaseDynamicStrategy design | S | Med | Design only — caseId classification |
| openclaw#40 | Config-driven layout (align with claudony) | S | Low | Follow-up from #38 |
