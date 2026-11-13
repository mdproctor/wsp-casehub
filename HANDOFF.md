# HANDOFF — casehub-examples (slot 196)

## Current Work

**Branch:** `issue-935-session-reuse-audit` (workspace), `issue-832-capacity-gaps` (neocortex), `main` (examples)
**Queue active:** casehubio/neocortex#845 — Upgrade CDE derivations to use PID-5 facets
**Queue file:** `.queue` with 7 tasks injected for #845

### Session Summary

Design session for #845. Closed two completed issues (#805, #832), then deep-dived into the PID-5/Jungian integration design with literature verification and pipeline architecture discovery.

### What Was Done

**#805 — Social emotion regulation — CLOSED:**
Already fully implemented across #806, #851, #902, #903. Closed with implementation summary comment.

**#832 — Capacity gap mapping — CLOSED:**
Created `neocortex/docs/research/capacity-gap-mapping.md` cataloging known emotion-personality capacity gaps across 5 subsystems (regulation, proactive, social, ToM, defence). Includes development checklist for new features.

**#845 — CDE PID-5 facet upgrade — DESIGN COMPLETE, PLAN WRITTEN:**

Major design findings:
1. **Pipeline fragmentation discovered.** AMPD/PID-5 facets are written to MindMap by `PersonalityProfileHandler` but NEVER read back. `EidosCognitiveProfileBridge` passes `List.of()` for personalityFacets. Five CDE methods that accept facets are dead code in production.
2. **Eidos/MindMap seam clarified.** Eidos = baseline personality (engram). MindMap = runtime personality deltas (shifts, degradation, growth). `PersonalityProfileHandler` was writing baseline data to the wrong store.
3. **Two-tier evidence model established.** Tier 1: peer-reviewed modulations (Impulsivity→WM, Callousness→empathy, Anxiousness→fear). Tier 2: first-principles extensions via FFM bridge (McCrae & Costa 1989 + Gore & Widiger 2013), documented with reasoning chains.
4. **PID-5 and Jungian are complementary, not competing.** PID-5 = WHAT traits. Jungian = HOW cognition works. Design extends rather than rewrites — Category A replaces DispositionAxes, Category B adds facet modulation to Jungian base.

Artifacts produced:
- Design spec: `wsp-casehub/specs/issue-845-cde-pid5-facets/2026-11-13-cde-pid5-facet-upgrade-design.md`
- Decisions (D1-D5): `wsp-casehub/specs/issue-845-cde-pid5-facets/decisions.md`
- Era 7 research doc: `neocortex/docs/research/era7-personality-trait-integration-research.md`
- Implementation plan: `wsp-casehub/plans/2026-11-13-cde-pid5-facet-upgrade.md`

**#712 — Shadow archetypes — UPDATED:**
Added implementation insights comment with Eidos/MindMap seam architecture, PID-5 facet delta representation for shadow degradation, and answers to all 4 open design questions.

### What's Next

**Implementation of #845** — follow the plan at `plans/2026-11-13-cde-pid5-facet-upgrade.md`:

1. **Batch 1: Pipeline foundation** — Extend `AgentDisposition` with `ampdFacets` (agency repo), fix `EidosCognitiveProfileBridge` to read them (neocortex). This is the prerequisite.
2. **Batch 2: Category A** — Upgrade 6 DispositionAxes methods to use `facetScore()`. Mechanical, follows `deriveSearchBudgetDefaults` reference pattern.
3. **Batch 3: Category B Tier 1** — 3 peer-reviewed modulations with citations.
4. **Batch 4: Category B Tier 2 + eval** — 5 FFM bridge modulations + integration test + eval validation.

### Key Design Decisions

- **D1:** Extend, not rewrite — PID-5 and Jungian measure complementary dimensions
- **D2:** Fix bridge (subsumed by D5)
- **D3:** Category A — PID-5 replaces DispositionAxes lookups
- **D4:** Category B — two-tier modulation (Tier 1 peer-reviewed, Tier 2 FFM bridge)
- **D5:** Unify pipeline — Engram → Eidos → Bridge → CDE, single path

### Cross-Repo Dependencies

- **Agency repo** must be modified first (Task 1 — AgentDisposition schema)
- **Neocortex** depends on the updated agency jar
- IntelliJ MCP required for implementation (was unavailable during planning session)

### Branch State

- Neocortex: `issue-832-capacity-gaps` — has era7 research doc + capacity gap mapping
- Workspace: `issue-935-session-reuse-audit` — has specs, decisions, plan, queue updates
- Examples: `main` (clean)
- Queue: 5 remaining issues after #845 (#921, #925, #938, #941)

### Fast Tests

Not run this session (design-only session, no code changes to CDE methods).
