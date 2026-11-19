# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-19c:** Completed #1160 (CAPS harness) and #1163 (test fix). #1162 blocked on JDK 21. Designed and filed era 8 divergence test suite (epic #1175, 7 child issues). Blog written. All published.

**Queue position:** 86/97 (11 open + 1 deferred). Next: #1172.

## What Was Done — Session 19c

### Issues Closed (code changes)

| SHA | Issue | Summary |
|-----|-------|---------|
| `e48396c52` | #1160 | `withBehavioralSynthesis(boolean)` + `capsSettlingResult()` on CognitionTestHarness — CAPS settling and behavioral attractor divergence test |
| `c88e8117b` | #1163 | EffectiveCognitiveDefaultsTest assertion fix — `"situational [usual 0.50]"` |

### Issues Blocked

| Issue | Blocker |
|-------|---------|
| #1162 | `maven.compiler.release=21`, ScopedValue needs JDK 25+ (#1173) |

### Issues Filed

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1171 | CAPS settling trajectory assertions | XS | Low |
| #1172 | CognitionTestHarness trajectory accessor | XS | Low |
| #1173 | Raise maven.compiler.release to 25+ | M | Med |
| #1174 | InnerLifeOrchestratorTest NPE | XS | Low |
| #1175 | Epic: cognitive pipeline divergence test suite | L | Med |
| #1176 | DivergenceFixtures + DerivationDivergenceTest | S | Low |
| #1177 | AppraisalDivergenceTest | S | Low |
| #1178 | ConsolidationDivergenceTest | S | Med |
| #1179 | PromptSectionDivergenceTest | S | Low |
| #1180 | TheoryOfMindDivergenceTest | M | Med |
| #1181 | EmotionRegulationDivergenceTest | M | Med |
| #1182 | Delete PersonalityDifferentiationTest | XS | Low |

### Research

- **Era 7** updated — #1160, #1163 landed; #1162 blocked
- **Era 8** created — cognitive pipeline validation research (48-capability audit, divergence test suite design, two-layer validation strategy)

### Blog

- `2026-11-19-mdp01-the-attractor-that-proved-personality.md` — CAPS behavioral attractor divergence testing

### Publishing

- neocortex: era7 update + era8 + #1160 + #1163 published to origin
- workspace: spec + blog + queue published to origin

## Queue — next items

| # | Title | Scale | Complexity | Depends on |
|---|-------|-------|-----------|-----------|
| #1172 | CognitionTestHarness trajectory accessor | XS | Low | — |
| #1174 | InnerLifeOrchestratorTest NPE | XS | Low | — |
| #1176 | DivergenceFixtures + DerivationDivergenceTest | S | Low | — |
| #1177 | AppraisalDivergenceTest | S | Low | #1176 |
| #1178 | ConsolidationDivergenceTest | S | Med | #1176 |
| #1179 | PromptSectionDivergenceTest | S | Low | #1176 |
| #1180 | TheoryOfMindDivergenceTest | M | Med | #1176 |
| #1181 | EmotionRegulationDivergenceTest | M | Med | #1176 |
| #1182 | Delete PersonalityDifferentiationTest | XS | Low | #1176-#1179 |
| #1173 | Raise maven.compiler.release to 25+ | M | Med | — |
| #1171 | CAPS settling trajectory assertions | XS | Low | — |

### Deferred

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1089 | Unbounded cache growth situational personality | XS | Low |

## Key Design Decision: Divergence Test Architecture

48 cognitive capabilities audited. 6 had divergence tests (12.5% coverage). Era 8 designs a systematic test suite:
- 7 test classes in `eval.divergence` package
- Shared `DivergenceFixtures` with OPEN/CAUTIOUS personality pairs
- Real pipeline (InMemoryCapsEngine, real CDE, real consolidation)
- All tests deterministic (no LLM)
- Spec: `wsp-casehub/specs/cognitive-divergence-test-suite/2026-11-19-divergence-test-suite-design.md`

## Key Finding: Tenant ID Trap

CAPS initialization in harness Builder used `"test-tenant"` while `consolidate()` runs with caller's tenant. Both agents silently auto-initialized with identical default axes. Fix: lazy initialization in `consolidate()` using caller's actual tenant ID.

## Neocortex Build Note

```bash
git -C neocortex fetch canonical main && git -C neocortex rebase canonical/main
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/pom.xml install -Dmaven.test.skip=true -s neocortex/.mvn/slot-settings.xml -T1C
```

## Known Pre-existing Test Failures

- `InnerLifeOrchestratorTest.tickWithAdditionalSectionsAppendsToSystemPrompt` — NPE, filed as #1174
