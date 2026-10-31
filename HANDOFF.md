# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-31
**Branch:** `issue-737-tiered-memory-architecture` (neocortex), `main` (examples)
**Issue:** neocortex#737 — Three-tier context memory (in progress, research phase)
**ERA:** Transitioning ERA5 → ERA6

## Status

**ERA5 complete.** Barrett v5 validated, 300-tick eval done, conclusion: "architecture, not tuning."

**ERA6 starting: Cognitive Memory Architecture.** Focus shifts from "can characters express personality convincingly" (proved in ERA5) to "can characters maintain coherent identity across context boundaries."

**Research phase complete for #737.** This session explored the tiered memory architecture through brainstorming, codebase audit, and literature research. Key finding: most of the infrastructure already exists — the gap is persistence, not computation.

## What This Session Did

### Tiered memory architecture design exploration

Through iterative brainstorming, arrived at a condensation-chain architecture:
- The MindMap IS the tiered memory — not a separate store
- Consolidation produces condensation chains (linked tiers), not in-place replacements
- Entity references at every tier — the query index
- Full transcript as immutable backing, never deleted
- Decay is retrieval priority modulation, not deletion
- Eviction from memory to disk, reconstructable on demand

### Codebase audit (5 subagent investigations)

| What was audited | Key finding |
|------------------|-------------|
| Agency summarisation | `LevelEvent` is entity-blind — no speaker/room/entity fields |
| Projections & folds | `ConversationFold` is already entity-aware; summarisation accumulators are not |
| Sub-thoughts | Already entity-referenced, graduation by (entity, type) pairs works |
| Chat-to-memory pipeline | Only crude one-line descriptions stored; full observation text discarded |
| Consolidation phases | 10 phases; merge, graduate, community detect all exist. No content condensation. |
| Per-prompt rendering | `TieredObservationRenderer` computes VERBATIM/GROUPED/SUMMARISED every tick — then discards it |

**Critical insight:** The system already computes tiered summaries on every tick and throws them away. The biggest gap is persistence, not computation.

### Literature research

**Primary reference:** TiMem (Li et al., 2026, ACL Findings, arXiv:2601.02845) — 5-level temporal memory tree. Validates the condensation approach.

Supporting: Spens & Burgess 2024 (reconstructive recall), ACT-R decay equations, CLS theory, MemGPT/Letta (OS-inspired tiers), PNAS 2022 hippocampal-neocortical model.

**Our additions beyond published work:** entity-indexed condensation chains, immutable transcript backing, cognitive-state-filtered reconstructive recall, two reset modes (sleep + context window).

### Commercial landscape

Mem0, Zep/Graphiti, Letta/MemGPT, MemPalace all shipping. None combines entity-indexed condensation chains with immutable backing and reconstructive recall.

### Commits

- `0df1e298` on `issue-737-tiered-memory-architecture` (neocortex) — research doc with full audit, literature, and design direction

## Two Reset Modes (Both Needed)

| | Sleep reset | Context reset |
|---|---|---|
| **When** | Narrative boundary, periodic | Mechanical, when context fills |
| **Injection** | Morning briefing — condensed history | Scene resume — who's here, what's happening |
| **Consolidation** | Async during sleep, Haiku | Must be ready before reset, continuous |
| **Tolerance** | High — "I vaguely remember..." | Zero — can't miss mid-sentence |

## Gap Table

| Gap | Severity | Notes |
|-----|----------|-------|
| Experience recording too thin | High | One-line descriptions, no entity extraction |
| Per-prompt rendering not persisted | High | Already computing, throwing away |
| Narrator summaries not stored | Medium | Already doing async LLM summarisation, discarding |
| No transcript persistence | High | Full observation text consumed and discarded |
| No content condensation phase | High | CommunitySummaryPhase is skeleton, needs LLM |
| No condensation chain edges | High | No `condensed-from` edge type |
| No persistent entity index | Medium | Per-run accumulation only |
| LevelEvent entity-blind | Medium | No speaker/room/entity fields |
| No context reset injection | High | New capability |

## Next Steps — ERA6 Roadmap

### Phase 1: Empirical test (next session)
1. Run a scenario for ~100 ticks
2. Snapshot what the memory system contains
3. Ask: "if I cleared context now, what would the character know?"
4. The gaps between need and reality become the real requirements

### Phase 2: Persist what's already computed
- Store per-prompt tiered rendering output as memory nodes
- Store narrator summaries as memory nodes
- Enrich experience recording with entity references

### Phase 3: Condensation chain
- Add `condensed-from` edge type to MindMap
- Extend CommunitySummaryPhase with LLM content condensation
- Build context reset injection from stored memories

### Phase 4: Full architecture
- Persistent entity index
- Decay and eviction policies (ACT-R-based)
- Reconstructive recall via sub-agents
- Two reset modes integrated

## Key Files

| What | Where |
|------|-------|
| **Research doc** | `neocortex/docs/research/tiered-memory-architecture-research-2026-10-31.md` |
| **Issue** | casehubio/neocortex#737 |
| **Barrett v5 instruction** | `examples/.../agent/CharacterCognition.java` lines 237-258 |
| **Smoke tests** | `examples/.../agent/PipelineToRenderingTest.java` → `goldenScenario*` |
| **Classifier** | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| **Golden prose** | `examples/wacky-manor/docs/eval/golden-prose-20261030/transcript.json` |

## Open Issues

| Issue | Repo | Title | ERA |
|:---:|------|-------|-----|
| #737 | neocortex | Three-tier context memory architecture | ERA6 |
| #183 | examples | Context reset with async summariser | ERA6 |
| #182 | examples | Integration test: P22 quality after consolidation | ERA5 carryover |
| #177 | examples | Comprehensive cognitive instrument as CI gate | ERA5 carryover |
| #178 | examples | Two-tier classification (Haiku→Opus) | ERA5 carryover |
| #179 | examples | DD formation echo weakness | ERA5 carryover |
| #734 | neocortex | Dynamic drive activation | ERA5 carryover |
| #735 | neocortex | Formation echo in dialogue-form | ERA5 carryover |

## Constraints

- Playbook, simulation data, manifest, and Eidos now available — makes empirical testing viable
- Do NOT remove cognitive sections — they represent abilities
- Isolation principle — one change at a time, measure, keep or revert
- Install neocortex before running examples
- Use subagents for generation/comparison work — keep main context clean
