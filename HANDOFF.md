# Session Handover — casehub-examples (slot 196)

**Date:** 2026-11-01
**Branch:** `issue-737-tiered-memory-architecture` (neocortex), `issue-191-persist-tiered-rendering` (examples)
**Issue:** neocortex#737 — Three-tier context memory architecture
**ERA:** ERA6 — Cognitive Memory Architecture

## START HERE — Orientation for Next Session

**What happened:** ERA6 Phase 2 is COMPLETE. Three persistence gaps closed — the system now stores observation summaries, narrator prose, and enriched experience records as MindMap nodes every tick.

**The core result:** Every tick, `ObservationService.drain()` persists the tiered rendering output (VERBATIM/GROUPED/SUMMARISED) as ACTIVITY nodes with room metadata. `NarratorAgent` stores LLM-generated scene narrations in a shared `narrator-summaries` subgraph. `Action` records now carry `room` and `targetAgent` as first-class fields promoted to reserved attributes in `toMemoryInput()`.

**Your immediate task is empirical validation:**
1. Run a 30-tick autonomous GENERIC scenario with the new persistence
2. Snapshot the MindMap store — verify observation nodes, narrator nodes, enriched experience attributes
3. Compare node counts and attribute quality against Phase 1 baseline (93 seeded, 31 runtime)
4. Assess whether the stored data is sufficient for context-reset injection (Phase 3)

**User design insight (captured, not yet implemented):** When rendering memories for context reset, provide a sequencing summary showing room transitions and significant events as a preamble to tie memories together temporally. Room data is stored per observation node to enable this. Needs empirical testing.

**After validation, Phase 3 candidates:**
- **#183** — Context reset with async summariser (condensation chain)
- Sequencing summary rendering for context-reset injection
- Node deduplication (#190, cosmetic, low priority)

**Key files modified in Phase 2:**
- `ObservationService` (examples) — `drain()` now calls `persistObservation()` with MindMapStore
- `NarratorAgent` (examples) — 8-param constructor stores narrations alongside broadcast
- `PlaybookOrchestrator` (examples) — wires MindMapStore into both ObservationService and NarratorAgent
- `Action` (neocortex memory-api) — `room`/`targetAgent` fields, backwards-compatible 9-param constructor
- `ExperienceEvents.toMemoryInput()` (neocortex) — promotes room/targetAgent to reserved attributes
- `ExperienceAttributeKeys` (neocortex) — `ROOM` constant added

**Do NOT:**
- Re-run Phase 1 empirical test — Phase 1 findings are documented and Phase 2 builds on them
- Fix #190 (node duplication) — cosmetic, defer to Phase 3
- Start condensation chain work (#183) without empirical validation of Phase 2 persistence

## Status

**ERA5 complete.** Barrett v5 validated, 300-tick eval done.

**ERA6 Phase 1 complete.** Empirical test, three bugs found, two fixed.

**ERA6 Phase 2 complete.** Three persistence issues closed:
- examples#191 CLOSED — tiered rendering persisted as ACTIVITY nodes
- examples#192 CLOSED — narrator summaries stored in shared subgraph
- neocortex#739 CLOSED — Action enriched with first-class room/targetAgent

**ERA6 Phase 3 next.** Condensation chains, context-reset injection, sequencing summary.

**Pre-existing test failure:** `PipelineToRenderingTest.barrettInstruction_includesStructuralGuidance` fails on main — unrelated to Phase 2 work.

## What This Session Did

### Phase 2 persistence implementation
- **#191:** `ObservationService.drain()` stores non-empty `ObservationResult` as ACTIVITY subgraph nodes per character (`observations-{characterId}`). Properties: content, tier, eventCount, timeSinceLastDrain, room. Provenance: `tiered-observation-renderer`.
- **#192:** `NarratorAgent` 8-param constructor stores narrations in shared `narrator-summaries` ACTIVITY subgraph. Backwards-compatible 6-param constructor preserved. Store call sits alongside existing WebSocket/Qhorus dispatch.
- **#739:** `Action` record gains `room` and `targetAgent` fields. `ExperienceEvents.toMemoryInput()` promotes both to reserved attributes when present. `AgentExperienceService.ingest()` uses new fields instead of metadata entries. `ROOM` constant added to `ExperienceAttributeKeys`.

## Commits This Session

**examples (issue-191-persist-tiered-rendering):**
- `e16e6fbb` — feat: persist tiered rendering output as MindMap memory nodes (#191)
- `bbc3e5ca` — feat: store narrator summaries as MindMap memory nodes (#192)
- `c8199349` — feat: use first-class room and targetAgent fields on Action (#739)

**neocortex (issue-737-tiered-memory-architecture):**
- `798eadba` — feat: enrich experience recording with entity references (#739)

## Open Issues

| Issue | Repo | Title | Status |
|:---:|------|-------|--------|
| #737 | neocortex | Three-tier context memory architecture | Phase 2 complete, Phase 3 next |
| #738 | neocortex | Domain mismatch in experience recall | FIXED |
| #739 | neocortex | Enrich experience recording with entity refs | CLOSED |
| #183 | examples | Context reset with async summariser | Phase 3 |
| #189 | examples | Scripted mode null experienceService | FIXED |
| #190 | examples | Node duplication from cache/seed | Open (deferred) |
| #191 | examples | Persist tiered rendering output | CLOSED |
| #192 | examples | Store narrator summaries | CLOSED |
| #177 | examples | Comprehensive cognitive instrument as CI gate | ERA5 carryover |
| #178 | examples | Two-tier classification (Haiku→Opus) | ERA5 carryover |
| #179 | examples | DD formation echo weakness | ERA5 carryover |
| #182 | examples | Integration test: P22 quality after consolidation | ERA5 carryover |
| #734 | neocortex | Dynamic drive activation | ERA5 carryover |
| #735 | neocortex | Formation echo in dialogue-form | ERA5 carryover |

## Constraints

- Install neocortex before running examples
- Use subagents for generation/comparison work — keep main context clean
- Isolation principle — one change at a time, measure, keep or revert
- Do NOT remove cognitive sections — they represent abilities
