# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-31
**Branch:** `issue-737-tiered-memory-architecture` (neocortex), `main` (examples)
**Issue:** neocortex#737 — Three-tier context memory architecture
**ERA:** ERA6 — Cognitive Memory Architecture

## START HERE — Orientation for Next Session

**What happened:** ERA6 Phase 1 is COMPLETE. Empirical test ran, three bugs found, two fixed. The memory infrastructure is stronger than expected — the gap is persistence, not computation.

**The core result:** 93 MindMap nodes seeded per character, 31 new nodes earned at runtime in 30 ticks. The system computes tiered summaries, narrator prose, and experience observations every tick — and throws away the first two. Experience memories ARE stored but couldn't be recalled (domain mismatch, now fixed). Scripted mode now wires experienceService (was null, now fixed).

**Your immediate task is Phase 2 — persist what's already computed:**

1. **examples#191** — Persist tiered rendering output as MindMap nodes. `TieredObservationRenderer` already computes VERBATIM/GROUPED/SUMMARISED every tick. Add a store call to save the output before discarding it.

2. **examples#192** — Store narrator summaries as memory nodes. `NarratorAgent` already produces LLM scene prose. Add a store call alongside the WebSocket broadcast.

3. **neocortex#739** — Enrich experience recording with entity references. Room, speaker, and target agent should be first-class attributes, not buried in metadata.

**Read before starting:**
- `neocortex/docs/research/era6-cognitive-memory-architecture-research.md` — updated with Phase 1 results, issue tracker, empirical gap table
- `examples/wacky-manor/docs/eval/era6-memory-snapshot/FINDINGS.md` — full empirical findings with data

**Key files for Phase 2:**
- `TieredObservationRenderer` (agency) — the renderer to tap for #191
- `ManorObservationRenderer` (examples) — wraps the agency renderer, runs compaction
- `NarratorAgent` / `NarratorSummariser` (examples) — the narrator pipeline for #192
- `ExperienceEvents.toMemoryInput()` (neocortex) — where entity enrichment goes for #739
- `AgentExperienceService` (examples) — now uses `ExperienceEvents.DOMAIN` for recall (fixed)

**Do NOT:**
- Re-run the empirical test — Phase 1 is done, findings documented
- Fix #190 (node duplication) — cosmetic, defer
- Start Phase 3 (condensation chains) — Phase 2 first

## Status

**ERA5 complete.** Barrett v5 validated, 300-tick eval done.

**ERA6 Phase 1 complete.** Empirical test, three bugs found, two fixed:
- neocortex#738 FIXED — domain mismatch in experience recall
- examples#189 FIXED — scripted mode null experienceService
- examples#190 OPEN — node duplication (cosmetic, deferred)

**ERA6 Phase 2 next.** Three issues filed, all depend on #738 (fixed).

## What This Session Did

### Empirical memory system assessment
- Built `MemorySnapshotResource` endpoint to query full MindMap + memory store state
- Ran 30-tick autonomous GENERIC scenario, captured snapshots at seed and post-run
- Discovered memory store is InMemoryMemoryStore (not NoOp as initially assumed)
- Quantified: 93 seeded nodes, 31 runtime nodes, 14 runtime memories stored

### Bug discovery and fixes
- **Domain mismatch (#738):** `ExperienceRecorderCore` stored in domain "experience", `AgentExperienceService.recall()` queried domain "manor". One-line fix: `MANOR_DOMAIN = ExperienceEvents.DOMAIN`. Regression test: `ExperienceRecallDomainTest`.
- **Scripted null (#189):** `PlaybookOrchestrator.runScripted()` passed `null` for experienceService. Fix: add parameter, pass real service. Regression test: `ScriptedModeExperienceWiringTest`.

### Blog entry
- "The Engine Was Already Running" — `wsp-casehub/blog/2026-10-31-mdp03-the-engine-was-already-running.md`
- Two SVG diagrams: compute-vs-persist flow, cognitive profile breakdown

### Issue filing
Filed 6 GitHub issues covering bugs (#738, #189, #190) and Phase 2 features (#191, #192, #739).

## Commits This Session

**examples (main):**
- `541528b6` — data(era6): Phase 1 empirical memory system assessment (#737)
- `cf05425d` — fix: align experience recall domain with storage domain (#738)
- `a7479273` — fix: wire experienceService into scripted mode (#189)

**neocortex (issue-737-tiered-memory-architecture):**
- `7c5e3314` — docs(era6): update research doc with Phase 1 results and bug fixes (#737)

**wsp-casehub (main):**
- `3736802` — wip(content): diary — The Engine Was Already Running

## Open Issues

| Issue | Repo | Title | Status |
|:---:|------|-------|--------|
| #737 | neocortex | Three-tier context memory architecture | Phase 1 complete, Phase 2 next |
| #738 | neocortex | Domain mismatch in experience recall | FIXED |
| #739 | neocortex | Enrich experience recording with entity refs | Phase 2 |
| #183 | examples | Context reset with async summariser | Phase 3 |
| #189 | examples | Scripted mode null experienceService | FIXED |
| #190 | examples | Node duplication from cache/seed | Open (deferred) |
| #191 | examples | Persist tiered rendering output | Phase 2 |
| #192 | examples | Store narrator summaries | Phase 2 |
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
