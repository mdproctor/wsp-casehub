# Session Handover — casehub-examples (slot 196)

**Date:** 2026-11-01
**Branch:** `issue-737-tiered-memory-architecture` (neocortex), `issue-191-persist-tiered-rendering` (examples), `issue-193-auto-register-observers` (agency)
**Issue:** neocortex#737 — Three-tier context memory architecture
**ERA:** ERA6 — Cognitive Memory Architecture

## START HERE — Orientation for Next Session

**What happened:** ERA6 Phase 3 complete. Three-layer memory architecture implemented: corpus ZIP transcripts (leaf), temporal compaction (derived), segment cache with context-reset injection (hot read). Blog entry written. Empirical validation is next.

**The core result:** Four commits implementing the three-layer architecture from the "Body Forgets" diagnosis. Raw transcripts persisted via ZipCorpusStore. Observation nodes bounded via temporal compaction during consolidation. Segment cache materialises observation history for injection into prompts as "What You've Experienced" section.

**Three-way tension hypothesis (untested):**
Context window clear threshold is a three-way optimisation: instruction adherence (wants short context), narrative continuity (wants long context), prompt caching (wants stable prefix). The three-layer architecture resolves all three — stable system prompt + slowly-changing segment cache = cacheable prefix; variable per-tick observation is small; instructions stay near generation point; continuity lives in memory not conversation history. The optimal clear point is hypothesised to be at the "knee" where instruction effectiveness drops sharply. Experiment needed: same scenario at 40k/60k/80k/120k thresholds.

**Immediate next steps for empirical validation:**
1. Run 100+ tick GENERIC scenario with context resets occurring naturally at 80k token threshold
2. Verify body vocabulary stays above 70% throughout, even across context clears
3. Verify transcript ZIP chain produces valid, readable archives
4. Verify observation compaction runs during consolidation and bounds node count
5. Verify segment cache text appears in prompts after context reset
6. Measure cache hit rate and latency impact of the stable-prefix architecture

**After validation:**
- Tune compaction parameters (window 10, batch 5) based on empirical data
- Test clear threshold at 40k/60k/80k/120k
- Watch for register conflict between segment cache (factual prose) and Barrett instruction (show don't tell)
- Consider generating cache text in character voice to avoid contamination

**Do NOT:**
- Re-implement Phase 3 — it's done, four commits on the branch
- Re-run Phase 2 empirical validation — done, data in `docs/eval/era6-memory-snapshot/`
- Fix #190 (node duplication) — cosmetic, defer
- Build condensed-from edges — Phase 3 uses erase-and-replace; linked tiers are Phase 4

## Status

**ERA5 complete.** Barrett v5 validated, 300-tick eval done.

**ERA6 Phase 1 complete.** Empirical test, three bugs found, two fixed.

**ERA6 Phase 2 complete + validated.** All three persistence features empirically confirmed:
- examples#191 CLOSED — tiered rendering persisted as ACTIVITY nodes
- examples#192 CLOSED — narrator summaries stored in shared subgraph
- neocortex#739 CLOSED — Action enriched with first-class room/targetAgent
- examples#193 CLOSED — observer auto-registration fix for GENERIC profile

**ERA6 Phase 3 complete.** Three-layer memory architecture implemented:
- Transcript persistence via corpus ZIP chain (leaf layer)
- Temporal compaction via ObservationCompactionPhase (derived layer)
- Segment cache with context-reset injection (hot-read layer)
- Blog: "Three Layers Deep" written and committed

**ERA6 Phase 4 next (deferred).** Condensation chain with linked tiers, persistent entity index, ACT-R decay, reconstructive recall.

**Pre-existing test failure:** `PipelineToRenderingTest.barrettInstruction_includesStructuralGuidance` fails on main — unrelated to ERA6 work.

## What This Session Did

### Phase 3 implementation (4 commits)

**Step 1 — Transcript persistence (fb724a3d):**
- `TranscriptStore` wraps `ZipCorpusStore` — appends `{tick, agentId, timestamp, prompt, response}` JSON per character per tick
- Virtual thread writes — off the tick critical path
- Added `casehub-neocortex-corpus` dependency to wacky-manor
- Config: `manor.transcript.enabled` (default true), `manor.transcript.storage-path` (default `~/claude/casehub/transcripts`)
- 5 tests in `TranscriptStoreTest`

**Step 2 — Temporal compaction (c7a54040):**
- `ObservationCompactionPhase` implements `ConsolidationPhase`, priority 50
- Per character `observations-<id>` subgraph: keeps last 10 nodes, groups older in batches of 5
- Concatenates content with `---` separators, tracks rooms, preserves time range
- `Condensed` trait for idempotency — subsequent passes skip condensed nodes
- Mechanical — no LLM calls. Follows feedback: prefer deterministic over LLM.
- Config: `manor.observation.compaction-recent-window` (10), `manor.observation.compaction-batch-size` (5)
- 6 tests in `ObservationCompactionPhaseTest`

**Step 3-4 — Segment cache + injection (a17de064):**
- `ObservationSegmentCache` reads condensed + recent observation nodes from MindMap
- Produces capped text summary per character (token budget = clearAtTokens/40 ≈ 2000 tokens)
- Formats "Earlier:" section from condensed nodes, "Recently:" from recent nodes, with room tags
- `ObservationBuilder.withObservationHistory(String)` adds "What You've Experienced" section
- Wired in `PlaybookOrchestrator`: cache created once, refreshed per tick, history injected into observation builder
- 6 tests in `ObservationSegmentCacheTest`

### Blog entry (b8051f2)
- "Three Layers Deep" — diary entry in wsp-casehub/blog/
- Two SVG diagrams: three-layer-implementation.svg, context-reset-bridge.svg
- Covers: problem (context windows aren't memory), three-layer solution, three-way tension (instruction adherence vs continuity vs prompt caching), empirical validation plan

### Repo sync
- Neocortex: local main synced to origin/main (6 new commits from #748, #749 — contextual modulation)
- Neocortex feature branch rebased onto updated main (clean, 4 commits)
- Agency: local main synced (5 new commits from #375, #376 — swarm coordination)
- Platform: local main synced (1 new commit)
- Connectors: slot-local Maven repo updated (stale 0.4-SNAPSHOT jar missing `Coordinates` class)
- Neocortex installed to slot-local Maven repo (knowledge-pipeline modules excluded due to pre-existing Coordinates package migration issue from #749)

## Commits This Session

**examples (issue-191-persist-tiered-rendering):**
- `fb724a3d` — feat: wire raw transcript persistence via corpus ZIP chain (#737)
- `c7a54040` — feat: temporal compaction for observation MindMap nodes (#737)
- `a17de064` — feat: observation segment cache with context-reset injection (#737)

**wsp-casehub (issue-191-persist-tiered-rendering):**
- `b8051f2` — wip(content): diary — Three Layers Deep Refs #737

**neocortex (issue-737-tiered-memory-architecture):**
- ERA6 research doc updated with Phase 2+3 completion

## Open Issues

| Issue | Repo | Title | Status |
|:---:|------|-------|--------|
| #737 | neocortex | Three-tier context memory architecture | Phase 3 complete, empirical validation next |
| #183 | examples | Context reset with async summariser | Segment cache wired; full async summariser deferred |
| #190 | examples | Node duplication from cache/seed | Open (deferred, cosmetic) |
| #177 | examples | Comprehensive cognitive instrument as CI gate | ERA5 carryover |
| #178 | examples | Two-tier classification (Haiku→Opus) | ERA5 carryover |
| #179 | examples | DD formation echo weakness | ERA5 carryover |
| #182 | examples | Integration test: P22 quality after consolidation | ERA5 carryover |
| #734 | neocortex | Dynamic drive activation | ERA5 carryover |
| #735 | neocortex | Formation echo in dialogue-form | ERA5 carryover |

## Key Files Modified This Session

**New files (examples):**
- `TranscriptStore.java` — corpus ZIP wrapper for transcript persistence
- `TranscriptStoreTest.java` — 5 tests
- `ObservationCompactionPhase.java` — consolidation phase for temporal compaction
- `ObservationCompactionPhaseTest.java` — 6 tests
- `ObservationSegmentCache.java` — materialised view for context-reset injection
- `ObservationSegmentCacheTest.java` — 6 tests

**Modified files (examples):**
- `PlaybookOrchestrator.java` — wired TranscriptStore, ObservationSegmentCache into tick loop
- `ObservationBuilder.java` — added `withObservationHistory(String)` for "What You've Experienced" section
- `ManorConfig.java` — added TranscriptConfig, extended ObservationConfig with compaction params
- `ManorConfigProducer.java` — wired transcript + compaction config properties
- `ManorConsolidationBeans.java` — produces ObservationCompactionPhase CDI bean
- `wacky-manor/pom.xml` — added casehub-neocortex-corpus dependency

## Hypotheses to Test

1. **Body vocabulary stability across resets:** Does body vocab stay above 70% when context clears at 80k and segment cache injects observation history? (Phase 2 showed 75%→51% degradation WITHOUT resets)

2. **Optimal clear threshold:** Is there a "knee" in the instruction effectiveness curve? Expected: sharp drop somewhere between 40k-80k tokens, with diminishing returns from longer context.

3. **Prompt cache alignment:** Does the stable-prefix architecture (system prompt + segment cache) measurably improve cache hit rate compared to growing conversation history?

4. **Register conflict:** Does factual segment cache text ("You were in the Library") contaminate Barrett instruction effectiveness ("show, don't tell")? Might need character-voice cache generation.

5. **Compaction parameter sensitivity:** Are window=10, batch=5 the right defaults? How does injection quality change if the window is 5 or 20?

## Constraints

- Install neocortex before running examples (exclude knowledge-pipeline modules: `-pl '!knowledge-pipeline-api,!knowledge-pipeline'`)
- Use subagents for generation/comparison work — keep main context clean
- Isolation principle — one change at a time, measure, keep or revert
- Do NOT remove cognitive sections — they represent abilities
- Sync slot repos to canonical mains at session start (feedback memory captured)
- Snapshot MindMap at tick 3-5 during scenario runs for early validation
- Save full scenario run transcripts to docs/eval/ before stopping the server
