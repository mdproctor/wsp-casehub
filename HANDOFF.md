# HANDOFF — casehub

**Date:** 2026-07-26
**Project:** `/Users/mdproctor/claude/casehub/examples`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**Wacky Manor POC — Phase 0 validated.** Brainstormed a multi-agent LLM demo
using Wacky Races characters in a haunted mansion. Built and validated Phase 0:
character behavior tests via Eidos descriptors, templates, and goals/constraints.
17/17 LLM eval tests pass — all five characters produce recognisable cartoon-grade
dialogue with expository soliloquy, plot device recognition, disguise maintenance,
and sustained multi-turn interaction.

Three platform features were built mid-session to support the POC:
- blocks#68: observation accumulator (tiered, RAG-able situation summaries)
- eidos#99: descriptor templates (reusable prose fragments)
- eidos#100: goals and constraints on AgentDescriptor

## Immediate Next Step

Start Phase 1 — game engine. Branch `feat/wacky-manor-poc` in `casehub-examples`
is open with Phase 0 complete. Run `/work` from the examples repo.

## Key Resources for Phase 1

The next session must read these before implementation:

| Resource | Path | What it contains |
|---|---|---|
| Vision | `examples/wacky-manor/docs/VISION.md` | Full scenario design — mansion, cast, plot beats, all ideas |
| POC Spec | `examples/wacky-manor/docs/POC-SPEC.md` | Phased spec with game engine design (§1.1–§1.7), UI (§2), reviewed and updated by adversarial design review |
| Implementation plan | `public/casehub/plans/2026-07-25-wacky-manor-poc.md` | Phase 0 tasks (done). Phase 1+2 deferred — needs new plan |
| Blog entry | `public/casehub/blog/2026-07-26-mdp01-wacky-manor-characters-come-alive.md` | Character output examples and architecture rationale |
| Memory | `~/.claude/projects/-Users-mdproctor-claude-casehub/memory/project_wacky-manor.md` | Project summary with key decisions |

## Phase 1 Scope (from POC-SPEC.md §1)

- §1.1 World model — rooms, objects, characters in YAML (`rooms.yaml`, `characters.yaml`)
- §1.2 ScenarioOrchestrator — single-threaded game loop with action queue, virtual threads per character
- §1.3 Action resolution — move, interact, take, give, use, look (deterministic, proximity-enforced)
- §1.4 Triggers and scenes — condition→effect pairs, scripted beat sequences with alternatives
- §1.5 Qhorus channels — `/manor/work` (dialogue), `/manor/audience` (narrator/asides), `/manor/oversight`
- §1.6 Narrator — per-scene-beat narration (full summarisation pipeline deferred to Phase 3)
- §1.7 Observation format — structured prompt template for character world view
- **blocks#68** — `ObservationAccumulator` for batching game events to LLM agents (tiered: verbatim/grouped/summarised)

## What's Left

- REPL explorer (Task 8) skipped — automated tests covered the same ground · XS · Low
- `CharacterProfile` and `CharacterProfileLoader` are now unused (superseded by Eidos runtime) — can be deleted · XS · Low

## Enabled

- `blocks` — observation accumulator (#68) shipped, ready for Phase 1 game engine integration
- `eidos` — templates (#99) and goals/constraints (#100) shipped and integrated in wacky-manor

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| — | Phase 1: game engine (world model + action loop + triggers + scenes) | L | Med | POC-SPEC.md §1.1–§1.7 fully designed |
| — | Phase 2: UI (mansion view + room chat columns + narrator panel) | M | Med | POC-SPEC.md §2.1–§2.6; Lit + Vite + Quinoa |
| — | Phase 3: full scenario (6 rooms, all characters, summarisation pipeline) | XL | High | Vision.md has full design |
