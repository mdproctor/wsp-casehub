# HANDOFF — casehub-examples

## Current Work

**Branch:** `issue-115-drive-expression-gap`
**Issue:** casehubio/examples#115 — Epic: Close the drive expression gap
**State:** Phase 1/2 inferred somatic rendering implemented (#120). Pipeline works end-to-end but scores regress from B0 due to somatic-only constraint. Next: TDD fixes for pipeline gaps + multi-channel expression.
**Plan:** `.plan` has #115 active (covers=115,118,119,120), #117 queued
**Epic:** examples#122 — tracks Batch 1 (get working) + Batch 2 (multi-channel)

## START HERE

**Research document (all findings, evidence, literature):**
`wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-research.md`
- **§14** — Phase 1/2 implementation results, bugs, root cause analysis (THIS SESSION)
- **§14.5** — Root cause of regression: somatic-only channel strips discrimination
- **§14.6** — Literature: four expression channels (somatic, action readiness, attentional, temporal)
- **§10.10.8** — Architecture: two-phase rendering (consolidation + render time)

## What Was Done This Session (2026-10-14, session 3)

### Implementation — #120 Inferred Somatic Rendering

**Phase 1 — Consolidation (neocortex):**
- `DriveDescriptionGenerationPhase` (P21) — LLM generates somatic descriptions from formation memories
- Generic fallback for drives without serving memories
- Somatic discrimination criteria for competing drives (reads DriveCompetitionInferred, generates via LLM)
- 12 unit tests

**Phase 2 — Render time (examples):**
- `CharacterCognition.renderInternalState()` reads DriveDescriptionGenerated, falls back to social-config
- SituationClassifier classifies current event → selects relevant formation memories
- "A memory surfaces — ..." priming in observation

### Bugs Found and Fixed (6)

| Bug | Impact | Fix |
|---|---|---|
| CDI proxy @Priority not inherited | Phases ran in random order — DriveDescription before ExperienceConsolidation | Walk superclass chain |
| maxPerPass=20 | Only 20 of 26 formation memories graduated — PP's 5 left out | Set to 50 |
| Scripted mode bypass | CharacterAgentLoop built observations without cognitive sections | Wire CharacterCognition |
| Significance race | Accumulator fired consolidateNow during seeding, locking out explicit call | Suspend/resume |
| discoverTenants security | Periodic consolidation ticks failed (cross-tenant admin) | Track known tenants |
| Mutiny atMost on virtual threads | Timeout fired internally but exception didn't propagate | Replace with Thread.join |

### Eval Results

**Micro tests:** PP gallantry 3/5 → 5/5 with full pipeline + somatic discrimination
**Full scenario (verified pipeline):** B0 mean 3.96 → 3.55 (regression)
- Above B0: HC scheming +0.13, HC self-preservation +1.26, AHM suspicion +0.85, Pen curiosity +0.19
- Collapsed: HC gloating -1.81, PP gallantry -1.57 (somatic-only doesn't discriminate)

### Root Cause Identified

The generation prompt says "somatic only." This produces descriptions that share vocabulary across competing drives. B0 hand-authored text discriminated through non-somatic channels (audience awareness, result-orientation, attentional narrowing). See §14.5.

### Issues Created

| Repo | Issue | What |
|---|---|---|
| examples | #121 | Multi-channel expression proposal |
| examples | #122 | Epic: get working + multi-channel |
| neocortex | #584 | DriveProfileDerivationPhase produces 0 profiles in scenario |
| neocortex | #585 | Generation prompt somatic-only strips discriminating channels |
| neocortex | #586 | Somatic discrimination text not generating |

### Diagnostic Infrastructure

| Tool | Purpose | Time |
|---|---|---|
| `/manor/cognitive-dump/{id}` | Live inspection: generated descriptions, profiles, competition, rendered observation | 1s |
| `ConsolidationPipelinePropagationTest` | Phases produce expected artifacts in order | 2.5s |
| `PipelineToRenderingTest` | Rendered observation contains right signals (6 assertions) | 4.7s |
| `DriveDescriptionDiscriminationTest` | Descriptions discriminate between drives | <1s |
| `ScriptedModeObservationTest` | Scripted mode includes cognitive sections | 4.4s |

### Cross-Repo Commits

**neocortex `issue-115-weighted-situation-vocabulary`:**

| Commit | What |
|---|---|
| `da40cf4c` | DriveDescriptionGenerationPhase — LLM somatic generation |
| `fab2fd26` | Somatic discrimination criteria for competing drives |
| `d5f47e3f` | Generic fallback for drives without memories |
| `ba249654` | Phase ordering — walk superclass chain for @Priority |
| `d99e3ac8` | Consolidation tick uses knownTenants instead of discoverTenants |
| `6d045be4` | Pipeline propagation tests |
| `17e19a9a` | Pipeline ordering tests |

**examples `issue-115-drive-expression-gap`:**

| Commit | What |
|---|---|
| `d8aa216` | renderInternalState reads generated descriptions |
| `c1bbf87` | Phase 2 situation priming via classifier overlap |
| `7164597` | Somatic discrimination criterion → 5/5 |
| `6216acb` | eventType "formation" → "formative" |
| `dad6ae1` | Suspend significance accumulator during seeding |
| `55cd3e5` | Replace Mutiny atMost with virtual thread join timeout |
| `512803f` | Wire cognitive sections into scripted mode |
| `df51351` | maxPerPass=50 |
| `1e141c5` | Cognitive dump diagnostic endpoint |
| `8ca8961` | Discrimination tests + verified run transcript |

## What Needs to Happen Next

### Priority 1: TDD the Pipeline Gaps (Batch 1, examples#122)

Each issue: reproduce with unit test, fix, verify test passes.

1. **neocortex#584 — DriveProfileDerivationPhase produces 0 profiles.** Cognitive dump shows 0 DriveProfileInferred for all characters. Without profiles: no competition detection, no differentiation, no situation priming. Likely a CDI injection issue with SituationClassifier/CapsTopology during consolidation.

2. **neocortex#586 — Discrimination text not generating.** Depends on #584. Once competition artifacts exist, verify the LLM discrimination generation works end-to-end.

3. **neocortex#585 — Generation prompt somatic-only.** The immediate fix: change "somatic only" to allow multi-channel descriptions. But do this AFTER #584/#586 so we can measure the combined effect.

4. **examples — Thinking prompt.** "Start there — in the body" reinforces somatic-only when the observation already constrains to somatic. Needs alignment.

5. **examples — Formation memory coverage.** Verify all 17 drives have serving memories. HC gloating was fixed this session; check for other gaps.

### Priority 2: Multi-Channel Expression (Batch 2, examples#121)

After Batch 1 is working:
1. Literature review: validate four-channel model (Barrett, Frijda, Easterbrook, Damasio)
2. Generation prompt: somatic → multi-channel
3. Situation-dependent channel selection via vocabulary classifier
4. Thinking prompt alignment

## Build Notes

**Neocortex must be on `issue-115-weighted-situation-vocabulary` branch.** Install before building wacky-manor:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/caps-api/pom.xml install -Dmaven.test.skip=true
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/caps-engine/pom.xml install -Dmaven.test.skip=true
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/mindmap-intelligence/pom.xml install -Dmaven.test.skip=true
```

**Run wacky-manor:**
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml quarkus:dev -Dquarkus.http.port=8180 -Dmanor.scenario.profile=generic
```

**After starting scenario, verify with cognitive dump:**
```bash
curl -4 -s http://127.0.0.1:8180/manor/start -X POST
# Wait 5 min for consolidation LLM calls
curl -4 -s http://127.0.0.1:8180/manor/cognitive-dump/peter-perfect | python3 -m json.tool
```

## Eval Baselines

| Metric | Value |
|---|---|
| B0 (BASELINE0 prescriptive) mean | 3.96 |
| Verified pipeline (somatic-only) mean | 3.55 |
| PP gallantry door (micro, full pipeline) | 5/5 |
| PP gallantry door (full scenario) | 2.93 |
| HC gloating (full scenario) | 2.16 (collapsed from 3.97) |
