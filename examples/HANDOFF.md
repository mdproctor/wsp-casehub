# HANDOFF — 2026-08-02

## Last Session

Personality composition research paper — wrote the full paper covering all 11 frameworks mapped to AgentDescriptor, the 5-axis coordinate system, 2 signal channels, compatibility matrix, experiment results, and extensibility. Corrected the results framing: the pipeline works (7-8/10 MBTI alignment), composition genuinely improves feeling/sensing activation (Penelope and Ant Hill Mob TAA 0.5→1.0), briefing dominance is a calibration problem not an architecture failure. Added a 6-point concrete engineering roadmap.

Closed #3 (12-run comparison) — all results are in. Created epic #11 with six child issues for the calibration gap work. The alignment gaps are small — coherence validation, judge calibration, and integration tuning, not architectural rework.

## Immediate Next Step

Phase 2.7 — live LLM narrator (issue #9, branch `issue-9-narrator-wiring`). Design spec written. `NarratorAgent` class exists but is not wired to the event stream.

## Completed This Session

- #10 CLOSED — research paper written and published on main
- #3 CLOSED — 12-run comparison done, results in the paper
- #11 created — epic for calibration gap with 6 child issues
- Paper pushed to both origin and upstream on main and feat/wacky-manor-poc

## Issue Status

| # | Title | State | Notes |
|---|-------|-------|-------|
| #3 | 12-scenario comparison | CLOSED | Results in the paper |
| #4 | Cheapest model alignment | OPEN | Not priority — wait |
| #9 | Phase 2.7 narrator wiring | OPEN | Current work, design spec done |
| #10 | Research paper | CLOSED | Published |
| #11 | Epic: calibration gap | OPEN | Small engineering tasks, not research |
| #12 | Briefing-framework coherence | OPEN | Highest value — catches Peter Perfect class of problem |
| #13 | Minimal briefing experiment | OPEN | Isolates framework contribution |
| #14 | Stronger integration | OPEN | Blocked by #13 |
| eidos#125 | Belbin axis implementation | OPEN | Bounded — mappings documented |
| eidos#126 | Judge calibration (Ni/Ne) | OPEN | May fix Ni activation immediately |
| eidos#127 | Full Big Five vocabulary | OPEN | Medium effort, well-documented mappings |

## What's Left (carried forward)

- blocks#76 spec + blog entry — deferred · S · Low
- blocks issue-47 rebased but not pushed · XS · Low
- Push garden entry GE-20260728-f7ad43 — pre-push hook blocked · XS · Low
- Wacky-manor not in parent pom modules list · XS · Low

## What's Next

| Phase | Description | Scale | Complexity | Notes |
|---|---|---|---|---|
| 2.7 | Live LLM narrator — wire NarratorAgent to accumulated event stream | S | Low | Design spec done, issue #9 |
| 2.8 | NPC system — Tier 2/3 scripted fixtures, player/NPC split | M | Med | RPG framing |
| 2.9 | Scale to 6 rooms — Library, Laboratory, Tower | L | Med | Re-evaluate buffer growth (#6) |
| 3.0 | Platform integration — memory, trust, HiL, replay | XL | High | Full casehub exercise |

## Cross-repo Changes This Session

| Repo | What | Branch |
|------|------|--------|
| casehubio/examples | Research paper + corrections | main + feat/wacky-manor-poc (pushed both remotes) |
| casehubio/examples | Issues #10, #11, #12, #13, #14 created; #3 closed | — |
| casehubio/eidos | Issues #125, #126, #127 created | — |

## Key Decisions

- Briefing dominance is a calibration problem, not architecture — the pipeline works, the gap is narrowing
- Composition is additive not redundant — each framework adds signal the LLM acts on
- Composite variance (6-13 turns) is a feature for creative scenarios, a cost for compliance
- Ni/Ne misclassification may be judge-side — worth validating before assuming framework limitation

## References

- Paper: `wacky-manor/docs/structured-personality-composition-in-llm-agents.md`
- Research brief: `work/examples/specs/issue-2-staged-layer-comparison/research-brief-personality-paper.md`
- Primary framework source: `eidos/docs/personality-frameworks.md`
- Narrator design spec: `work/examples/specs/issue-9-narrator-wiring/2026-08-02-narrator-wiring-design.md`
- Phase 2.5 spec: `wacky-manor/docs/specs/2026-07-27-phase-2.5-autonomous-characters-design.md`
- Deferred issues: #4 (cheapest model), #6 (buffer growth), #7 (per-type EventLevels), #8 (full blocks pipeline)
