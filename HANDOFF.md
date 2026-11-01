# Session Handover — casehub-examples (slot 196)

**Date:** 2026-11-01
**Branch:** `issue-737-tiered-memory-architecture` (neocortex), `issue-191-persist-tiered-rendering` (examples), `issue-193-auto-register-observers` (agency)
**Issue:** neocortex#737 — Three-tier context memory architecture
**ERA:** ERA6 — Cognitive Memory Architecture

## START HERE — Orientation for Next Session

**What happened:** ERA6 Phase 2 empirically validated. All three persistence features work. One bug found and fixed (#193 — observer auto-registration). Phase 3 design direction established.

**The core result:** 30-tick GENERIC scenario confirmed narrator summaries (9 nodes) and experience enrichment (room 85%, target-agent 44%). Observation persistence was broken under GENERIC profile due to observer registration using pre-remap IDs. Fixed in agency-core — `PartitionedObservationService.publishEvent()` now auto-registers observers via `computeIfAbsent`. Validated with 5-tick re-run: 5 observation subgraphs, 4 nodes each.

**Phase 3 design direction — three-layer memory architecture:**

| Layer | Store | Content | Role |
|-------|-------|---------|------|
| **Leaf** | Corpus ZIP chain (`neocortex/corpus/zip`) | Raw chat transcripts (prompt + response per tick) | Archive, re-derivation, audit |
| **Derived** | MindMap ACTIVITY nodes | Rendered observations (tiered), narrator summaries | Source of truth for summaries |
| **Hot read** | Segment cache (new) | Materialised view over MindMap nodes | Fast injection on context reset |

**Key design decisions:**
1. **Corpus ZIP chain for transcript preservation** — `ZipCorpusStore` with rollover + `Compactor` already exists in `neocortex/corpus/zip`. Raw transcripts are the leaf data from which all derived layers come. Append per-tick, rollover at size threshold, compact old archives.
2. **MindMap nodes stay for derived summaries** — observation nodes contain the tiered rendering output (a lossy transformation), not just raw data. Can't be cheaply reconstructed from transcripts. MindMap is the source of truth for the rendered/summarised content.
3. **Temporal compaction for MindMap observation nodes** — nodes grow linearly (~4/character/5-ticks). Need a compaction phase that merges older observation nodes into condensed summary nodes with `condensed-from` edges. Similar to `CommunitySummaryPhase` but temporal instead of graph-structural.
4. **Lazy segmented cache for injection** — materialise "last N observations" + "condensed history" on demand from MindMap nodes. Cache segments by tick range, extend incrementally when new observations arrive. Context-reset injection reads from cache, never from corpus ZIP.

**Immediate next steps for Phase 3:**
1. Wire raw transcript persistence via corpus ZIP chain in `PlaybookOrchestrator` — append prompt+response per character per tick
2. Design temporal compaction phase for observation MindMap nodes — bounded window + merge older nodes
3. Build segment cache over MindMap observation nodes for context-reset injection
4. Wire injection into session context-reset path (triggered at 80k tokens via `manor.session.clear-at-tokens`)

**Context-reset injection budget (from 30-tick empirical data):**
- 34 experience memories per character at ~536 chars each = ~4562 tokens total
- Last 10 memories = ~1099 tokens (fits budget)
- Condensed summary of older memories = ~1368 tokens estimated
- Room-transition preamble = ~100 tokens
- Available budget for injection: ~2000 tokens

**After Phase 3, remaining candidates:**
- Node deduplication (#190, cosmetic, low priority)
- Sequencing summary rendering for context-reset preamble

**Key files modified this session:**
- `PartitionedObservationService` (agency-core) — auto-register observers in `publishEvent()` via `computeIfAbsent`
- `ObservationService` (examples) — removed pre-registration loop from `init()`
- `ObservationServiceTest` (examples) — 2 new tests for ID remap scenario

**Do NOT:**
- Re-run Phase 2 empirical validation — done, data in `docs/eval/era6-memory-snapshot/`
- Fix #190 (node duplication) — cosmetic, defer
- Build the segment cache without first wiring transcript preservation — leaf layer comes first

## Status

**ERA5 complete.** Barrett v5 validated, 300-tick eval done.

**ERA6 Phase 1 complete.** Empirical test, three bugs found, two fixed.

**ERA6 Phase 2 complete + validated.** All three persistence features empirically confirmed:
- examples#191 CLOSED — tiered rendering persisted as ACTIVITY nodes
- examples#192 CLOSED — narrator summaries stored in shared subgraph
- neocortex#739 CLOSED — Action enriched with first-class room/targetAgent
- examples#193 CLOSED — observer auto-registration fix for GENERIC profile

**ERA6 Phase 3 next.** Transcript preservation (corpus ZIP), temporal compaction, segment cache, context-reset injection.

**Pre-existing test failure:** `PipelineToRenderingTest.barrettInstruction_includesStructuralGuidance` fails on main — unrelated to ERA6 work.

## What This Session Did

### Empirical validation (30-tick GENERIC run)
- Confirmed narrator persistence (#192): 9 nodes in `narrator-summaries` subgraph
- Confirmed experience enrichment (#739): room 85%, target-agent 44% of memories
- Found observation persistence (#191) broken under GENERIC profile — 0 observation subgraphs

### Bug fix (#193): observer auto-registration
- Root cause: `ObservationService.init()` pre-registered observers with original IDs; GENERIC `remapCharacterIds()` changed all IDs; events silently dropped
- Fix: `PartitionedObservationService.publishEvent()` auto-registers via `computeIfAbsent`; pre-registration removed from `ObservationService.init()`
- Validated with 5-tick GENERIC re-run: 5 observation subgraphs, 20 nodes total

### Repo sync
- Examples canonical main synced with 61 ERA5+ERA6 commits
- Neocortex slot rebased onto canonical main (incorporated 7 #736 commits), feature branch rebased (4 commits)
- Agency fix committed on `issue-193-auto-register-observers`, merged to agency main

### Context-reset injection assessment
- 34 experience memories per character (~4562 tokens raw, ~1099 for last 10)
- Room transitions reconstructible from stored attributes (6 rooms visited)
- Interaction partner tracking working (3 partners, frequency-ranked)
- Identified three-layer architecture: corpus ZIP (leaf) → MindMap nodes (derived) → segment cache (hot read)

## Commits This Session

**examples (issue-191-persist-tiered-rendering):**
- `a80608bf` — wip: stage Phase 1 eval snapshots (penelope, classification audit)
- `0b17c92c` — data(era6): Phase 2 empirical validation — 30-tick GENERIC run (#175)
- `5a112667` — fix: remove pre-registration from ObservationService.init (#193)
- `ab403b1d` — data(era6): Phase 2 validation — observation persistence confirmed (#193)

**agency (issue-193-auto-register-observers, merged to main):**
- `ab7a3488` — feat: auto-register observers in PartitionedObservationService (#193)

## Open Issues

| Issue | Repo | Title | Status |
|:---:|------|-------|--------|
| #737 | neocortex | Three-tier context memory architecture | Phase 2 validated, Phase 3 design ready |
| #183 | examples | Context reset with async summariser | Phase 3 — needs transcript preservation first |
| #190 | examples | Node duplication from cache/seed | Open (deferred, cosmetic) |
| #193 | examples | Observer auto-registration for GENERIC profile | CLOSED |
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
- Sync slot repos to canonical mains at session start (feedback memory captured)
- Snapshot MindMap at tick 3-5 during scenario runs for early validation
