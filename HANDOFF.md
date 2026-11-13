# HANDOFF — casehub-examples (slot 196)

## Current Work

**Branch:** `issue-935-session-reuse-audit` (workspace), `issue-845-cde-pid5-facets` (neocortex), `main` (examples)
**Queue active:** casehubio/neocortex#845 — Upgrade CDE derivations to use PID-5 facets
**Queue file:** `.queue`

### Session Summary

Implementation session for #845. Completed Tasks 3-7 (all CDE derivation method upgrades + tests). Tasks 1-2 deferred — `AgentDisposition` is in the eidos repo (not in this slot), not the agency repo as the plan assumed.

### What Was Done

**#845 — CDE PID-5 facet upgrade — TASKS 3-7 COMPLETE:**

Commit `504f1c91` on `issue-845-cde-pid5-facets` branch in neocortex.

**Category A — 6 axes methods upgraded to accept PID-5 facets:**
- `deriveMoodBaseline`: Risk Taking→pleasure, Emotional Lability→arousal, Withdrawal→dominance
- `deriveCuriosity`: Withdrawal→structural, Impulsivity→quality, Risk Taking→centrality
- `deriveCbrStrategy`: Impulsivity→minSimilarity, Risk Taking→decayDays
- `deriveSocialCognition`: Suspiciousness→trustRate
- `deriveHabituationConfig`: Impulsivity→habRate/repTol, Risk Taking→novelty
- `deriveRegulationDefaults`: Emotional Lability→reappraisal, Callousness/Withdrawal→social

**Category B Tier 1 — 3 peer-reviewed modulations:**
- `deriveCapacity`: Impulsivity→WM (Heydarloo 2024, Kofler 2024)
- `deriveExtractionBias`: Callousness→relationship bias (Blair 2005/2008/2013)
- `deriveAppraisalWeights`: Anxiousness→fear (Bar-Haim 2007)

**Category B Tier 2 — 5 FFM bridge modulations (McCrae & Costa 1989, Gore & Widiger 2013):**
- `deriveExtractionBias`: Emotional Lability→affect sensitivity
- `deriveAppraisalWeights`: Risk Taking→fear, Impulsivity→urgency
- `deriveCapacity`: Anxiousness→associative reach
- `deriveGraphStructure`: Withdrawal→connective bias

**Integration test:** Full pipeline test with combined Jungian + AMPD facets.
**Backward compat:** Added overloads for `deriveRegulationDefaults(axes)`, `deriveCapacity(profile)`, `deriveAppraisalWeights(profile, axes)` to avoid breaking 20+ existing callers.

**Test results:** 23 new tests + 440 total cognitive-index tests + 7 cognitive-eidos bridge tests = 0 failures.

### Plan correction — repo mismatch

The plan assumed `AgentDisposition` lives in the **agency** repo (`agency/eidos-api/`). It actually lives in the **eidos** repo (`/Users/mdproctor/claude/casehub/eidos/api/`). The eidos repo is not in slot 196.

This means:
- Task 1 (extend `AgentDisposition` with `ampdFacets`) → requires eidos repo
- Task 2 (fix `EidosCognitiveProfileBridge` to read AMPD facets) → depends on Task 1

Both are deferred. The CDE methods are fully upgraded and tested — they accept `List<PersonalityFacet>` via `DescriptorView`. When Tasks 1-2 are done, facets will flow from Eidos through the bridge automatically with no further CDE changes.

### What's Next

1. **Task 1:** Add `ampdFacets` field to `AgentDisposition` in the eidos repo. Requires either adding the eidos repo to slot 196 or working in the canonical copy.
2. **Task 2:** Fix `EidosCognitiveProfileBridge.toDescriptorView()` to map `AmpdFacet` → `PersonalityFacet`. Currently passes `List.of()` for facets.
3. **Eval validation:** Run wacky-manor fast tests to verify downstream integration.
4. **Remaining queue:** #921, #925, #938, #941

### Key Design Decisions

- **D1:** Extend, not rewrite — PID-5 and Jungian measure complementary dimensions
- **D2:** Fix bridge (subsumed by D5) — DEFERRED (eidos repo not in slot)
- **D3:** Category A — PID-5 modulates DispositionAxes lookups — DONE
- **D4:** Category B — two-tier modulation (Tier 1 peer-reviewed, Tier 2 FFM bridge) — DONE
- **D5:** Unify pipeline — Engram → Eidos → Bridge → CDE, single path — PARTIALLY DONE (CDE ready, bridge deferred)

### Branch State

- Neocortex: `issue-845-cde-pid5-facets` — all CDE method upgrades + 23 new tests
- Neocortex: `issue-832-capacity-gaps` — has era7 research doc + capacity gap mapping (not merged to main)
- Workspace: `issue-935-session-reuse-audit` — has specs, decisions, plan, queue updates
- Examples: `main` (clean)

### Fast Tests

- cognitive-index: 440 tests, 0 failures (full suite)
- cognitive-eidos: 7 tests, 0 failures (bridge module)
- wacky-manor fast tests: not yet run this session
