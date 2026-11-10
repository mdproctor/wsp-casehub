# HANDOFF — casehub (slot 196)

## Last Session (2026-11-10)

Fixed #906 (topology weight rebalancing), #907 (DISTRESS intensity gating), #912 (accommodating → CC boost). Ran eval twice — baseline showed 0/5, rebalanced showed 3/5. Four-agent adversarial review validated dropping self_worth → AD connection (indirect path captures 75% of empirical effect). Fixed pre-existing RegulationPipelineThroughTickTest failures (3 tests — stub overrode wrong generate overload). Wrote blog: "Every Weight Has a Citation". Updated post-era6 research document.

### #906 — Weight rebalancing

4 literature-justified topology changes:
1. Added `self_efficacy → AD` at −0.25 (Sheppes & Gross 2011)
2. Reduced `arousal → AD` from +0.45 to +0.35 (midpoint of meta-analysis range)
3. Reduced `self_worth → RM` from −0.25 to −0.15
4. Reduced `self_efficacy → RM` from −0.25 to −0.15
Plus 50% increase in disposition resting activation differentials.

| SHA | Message |
|-----|---------|
| `4cd9ee62` | fix(#906): rebalance CAPS regulation topology weights |
| `90289815` | docs(#906): save rebalanced CAPS regulation eval results |

### #907 — DISTRESS intensity gating

| SHA | Message |
|-----|---------|
| `36a21ca5` | fix(#907): intensity-gate DISTRESS in predictor |

## Previous Session (2026-11-09)

### Task 8 commit

| SHA | Message |
|-----|---------|
| `0c880cf9` | feat(#806): integrate CAPS settling into regulation cascade |

### Task 8 details

- **Safety hard gate**: fatigue > 0.7 OR intensity > 0.75 → force ATTENTIONAL_DEPLOYMENT before CAPS/predictor
- **CAPS settling**: loads agent state, assembles social-context inputs (trust from MindMap, isolation from agent count), settles, extracts regulation attractors with dominance margin check
- **Predictor fallback**: existing predictor runs only when CAPS doesn't pin
- **Weight update**: Rescorla-Wagner after regulation — outcomeIntensity = |peak - regulated|, outcomeValence = effectiveness > 0.5 → +1.0 else -1.0, salienceMultiplier = peak, schedule = variable_ratio
- **CognitionCore wiring**: new `configureRegulation(CapsEngine, RegulationCascadeConfig)` overload added
- **Tests**: 4 cascade tests (pin, ambivalent fallthrough, safety gate, weight update) — 72 regulation tests total, all pass

### Task 9 commit

| SHA | Message |
|-----|---------|
| `4d390d0f` | feat(#806): add CAPS regulation comparison eval test |

### Task 9 details

- `CapsRegulationComparisonEvalTest` — tagged `@Tag("llm-eval")`, runs 5 per-strategy scenarios
- Each scenario: cascade tick (CAPS → predictor → LLM) + LLM-only tick + standalone CAPS settling + standalone predictor
- Agreement summary: CAPS-predictor, CAPS-LLM, predictor-LLM concordance rates
- Threshold sweep: concordance at thresholds 0.1–0.5 for empirical calibration
- **Ran successfully** — results saved to `docs/eval/caps-regulation-comparison-baseline.json`
- Agreement: 60% across all three pairs (CAPS-predictor, CAPS-LLM, predictor-LLM)
- CAPS never pins (margin 0.052 < 0.15 dominanceMargin) — falls through to predictor in all 5 scenarios
- Two calibration issues: AD structural weight advantage in topology, DISTRESS→UNCONTROLLABLE_NEGATIVE mapping too coarse

### Next session

Branch `issue-806-caps-regulation-integration` pushed (16 commits). All calibration issues resolved (#906, #907, #912). `work continue` to advance.

**Immediate:** Rerun `CapsRegulationComparisonEvalTest` to verify #912 (accommodating → CC) improves eval from 3/5 to 4/5 or 5/5. Then advance to next queue item.

**Queue (next unchecked):**
- #913 — Protective conflictMode regulation mapping (S/Med — investigate whether SM is correct for protective agents)
- #894 — Pipeline coherence eval (requires #520 cognitive state dump — different domain)
- #831 — Personality coherence check (S/Low)
- #832 — Capacity gaps across pipeline (S/Low)

**Also filed:** #912 is fixed but the eval hasn't been rerun with it. The accommodating → CC boost should fix the cognitive change scenario (CAPS was settling to SSS because "accommodating" had no CC resting activation).

**Pre-existing fix included:** RegulationPipelineThroughTickTest (3 failures — stub overrode 7-param generate but tick calls 11-param). Fixed in same commit as #912.

### What was implemented

**Branch:** `issue-806-caps-regulation-integration` in neocortex (7 commits)

**Batch 1 — Foundation (Tasks 1-2):**
- Renamed `BehavioralAttractor` → `CapsAttractor` across 11 files (IDE refactoring, 45 usages)
- Added `RegulationCascadeConfig` record (regulationConfidenceThreshold=0.3, regulationDominanceMargin=0.15) in `cognition-api`

**Batch 2 — Topology (Tasks 3-4):**
- Extended `caps-topology.yaml`: 14 new nodes (10 regulation outputs + 4 social-context inputs), 34 new connections. Topology now: 91 nodes, 110 connections
- Added regulation resting activations to `DispositionWeightMapper` (avoidant→situation_selection, analytical→cognitive_change, etc.)
- Fixed `"confrontational"` → `"competitive"` inconsistency in conflictMode

**Batch 3 — Namespace (Task 5):**
- Added `registerParticipantGoals(participantId, agentId, tenantId, goals)` to `GoalProposalOrchestrator`
- Migrated `EmotionRegulationGoalParticipant` and `ProactiveRegulationParticipant` from `registerGoals`/`appendGoals`

**Batch 4 — Social + Decay (Tasks 6-7):**
- Created `SocialContextInputComputer` with 4 literature-grounded formulas (perceived_responsiveness, target_distress, social_isolation, power_asymmetry). Pure static methods for formula computation; MindMap data extraction deferred to Task 8 integration.
- Replaced linear decay with exponential in `ProactiveRegulationParticipant` (decayHorizonTicks → decayTimeConstant, Math.exp formula)

### Commits

| SHA | Message |
|-----|---------|
| `90270409` | refactor(#806): rename BehavioralAttractor → CapsAttractor |
| `4d3059d7` | feat(#806): add RegulationCascadeConfig for cascade pin decisions |
| `aba16fe0` | feat(#806): extend CAPS topology with regulation + social-context nodes |
| `81b6df32` | feat(#806): add regulation resting activations to DispositionWeightMapper |
| `7ab0cb7b` | feat(#806): add namespace isolation to GoalProposalOrchestrator |
| `39ee856e` | feat(#806): exponential decay for proactive avoidance |
| `868c5b5f` | feat(#806): add SocialContextInputComputer for CAPS regulation inputs |

### Tests added

- `RegulationCascadeConfigTest` — 3 tests (defaults, negative validation)
- `RegulationTopologyTest` — 7 tests (node existence, connection verification, count assertions)
- `RegulationDispositionMappingTest` — 8 tests (resting activations per disposition axis)
- `SocialContextInputComputerTest` — 15 tests (all 4 formulas + edge cases)
- 3 namespace isolation tests added to `GoalProposalOrchestratorTest`
- 1 exponential decay test added to `ProactiveRegulationParticipantTest`

### Next session

`work continue` → executing-plans, resume at **Task 8: Cascade integration in EmotionRegulationTickParticipant**.

Task 8 is the core deliverable — integrating CAPS settling into the regulation cascade. Key files:
- `EmotionRegulationTickParticipant.java` — main modification target (tick method)
- `CognitionCore.java` — wiring CapsEngine and SocialContextInputComputer
- `CapsEngine.java` — SPI for settle() and updateWeights()
- Design spec: `wsp-casehub/specs/issue-851-dynamic-regulatory-capacity/2026-11-09-caps-regulation-integration-design.md` §Strategy Selection Cascade and §Integration with Existing Pipeline

Implementation steps (from plan):
1. Write cascade test — CAPS pins strategy
2. Write cascade test — CAPS ambivalent, predictor fallback
3. Write cascade test — safety hard gate
4. Write cascade test — CAPS weight update after regulation
5. Run tests — verify they fail
6. Implement cascade in EmotionRegulationTickParticipant.tick()
7. Wire CapsEngine and SocialContextInputComputer into CognitionCore
8. Run tests — verify they pass
9. Run full regulation unit test suite for regression
10. Commit

After Task 8: Task 9 (paired comparison eval test, requires API key).

### MindMap API notes (from research this session)

For Task 8 — social-context data extraction from MindMap:
- **Trust**: `RelationalSchema.fromProperties(overlay.properties()).trust()` [-1,1]. Normalize to [0,1] with `(trust+1)/2` before passing to `SocialContextInputComputer.computeResponsiveness()`
- **PAD**: `MindMapNode.pleasure()`, `.arousal()`, `.dominance()` — all `Double` nullable
- **Overlay query**: find PERSON subgraph → find target node by agentId → find overlay with `traits.contains("overlay")` and matching observer agentId
- **RegulationTracker**: `RegulationTracker.computeTrajectory(targetId)` → `Optional<TrajectoryResult>` with `.trajectory()` enum (IMPROVING, STABLE, WORSENING)

---

## Previous Session (2026-11-09)

Designed CAPS regulation integration (#806). Queue reorganised to focus on regulation work only — #203 (spatial) and #315 (dashboard) removed.

### What was designed

**#806 — CAPS regulation integration: strategies as CAUs**

Full design spec and implementation plan for encoding all 10 regulation strategies as CAPS output nodes with 4 literature-grounded social-context input nodes. Three-layer cascade (CAPS > predictor > LLM). Design went through brainstorming (8 decisions D20-D27), decision review (2 rounds, 12 issues, all resolved), spec writing, and spec review (3 rounds, 17 issues, all resolved).

### Artifacts produced

| Artifact | Path |
|----------|------|
| Design spec | `wsp-casehub/specs/issue-851-dynamic-regulatory-capacity/2026-11-09-caps-regulation-integration-design.md` |
| Decisions D20-D27 | `wsp-casehub/specs/issue-851-dynamic-regulatory-capacity/decisions.md` |
| Implementation plan | `wsp-casehub/plans/2026-11-09-caps-regulation-integration.md` |
| Blog entry | `wsp-casehub/blog/2026-11-09-mdp07-when-regulation-joins-the-network.md` |

## Queue — Remaining Work

### Epic #821 — Emotion regulation ecosystem

| # | Title | Scale | Complexity | Status |
|---|-------|-------|------------|--------|
| #806 | CAPS integration — strategies as CAUs | L | High | In progress (7/9 tasks done) |
| #831 | Personality coherence check across all mappings | S | Low | Open |
| #832 | Emotion-personality capacity gaps | S | Low | Open |
| #844 | Regulation prediction feedback loop | — | — | Open |
| #850 | Split eval: congruence vs effectiveness | S | Med | Open |
| #894 | Pipeline coherence eval | — | — | Open |

### Epic #851 — Dynamic regulatory capacity

| # | Title | Scale | Complexity | Status |
|---|-------|-------|------------|--------|
| #853 | Capacity modulation function | S | High | Open |
| #854 | Timescale-aware modulation weighting | S | Med | Open |
| #855 | End-to-end validation with compounding shame | S | Med | Open |

### Deferred

| # | Title | Scale | Complexity |
|---|-------|-------|------------|
| #902 | Distress trajectory enrichment | S | Med |
| #903 | Social regulation toggle | XS | Low |
| #904 | Social integration test | S | Low |

## What to Watch

1. **Branch not pushed.** 7 commits on `issue-806-caps-regulation-integration` in slot 196 neocortex. Not pushed to origin yet — push after Task 8 completes and passes full test suite.

2. **Pre-existing build failures.** `schema-generator` and `knowledge-pipeline` still fail on canonical main. Unrelated.

3. **SocialContextInputComputer is pure formulas only.** MindMap data extraction (overlay query, trust lookup, PAD extraction) is deferred to Task 8 — it belongs in EmotionRegulationTickParticipant where the tick context is available.
