# LLM Eval Tests for Cognitive Stack — Design Spec

**Issue:** casehubio/examples#59
**Epic:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Date:** 2026-09-15
**Status:** Draft

---

## Problem

The cognitive observation sections (drives, beliefs, norms) are wired and rendered, but there's no verification that they actually influence LLM character behavior. Unit tests verify the sections appear in the observation — they don't verify the LLM reads and acts on them.

---

## Design

### Test Architecture

One test class — `CognitiveInfluenceEvalTest` — with `@ParameterizedTest` driven by `CognitiveScenario` records. Each scenario:

1. Loads the character's system prompt via `SystemPromptRenderer`
2. Constructs the cognitive observation via `CharacterCognition.renderCognitiveSections()`
3. Prepends the observation to a scenario-specific user prompt
4. Invokes the LLM via `AgentInvocationService` → gets `AgentResponse`
5. Invokes a second LLM call (judge) with the response text and a scoring prompt
6. Asserts the judge score meets a minimum threshold

```
@QuarkusTest @Tag("llm-eval")
CognitiveInfluenceEvalTest
  ├── @ParameterizedTest @MethodSource("scenarios")
  │   └── cognitiveStateInfluencesBehavior(CognitiveScenario)
  │       ├── render system prompt
  │       ├── build observation with cognitive sections
  │       ├── invoke LLM (agent)
  │       ├── invoke LLM (judge)
  │       └── assert score >= threshold
  └── record CognitiveScenario(name, agentId, situationPrompt,
  │                              judgeCriteria, cognitiveElement)
  └── scenarios() → Stream<Arguments>
```

### Scenarios

| # | Name | Character | Cognitive Element | Situation | Judge Criteria |
|---|------|-----------|------------------|-----------|---------------|
| 1 | scheming-drive | hooded-claw | Drive: scheming 90% | "You overhear Penelope discussing a valuable item. Peter Perfect just left the room." | "Does the response show scheming behavior — planning, manipulation, or indirect tactics rather than direct confrontation?" |
| 2 | trust-norms | penelope-pitstop | Norm: "Trust everyone until proven otherwise" | "A stranger arrives at the manor claiming to need help finding a lost item." | "Does the response show trusting behavior — willingness to help, assuming good intentions, not questioning motives?" |
| 3 | belief-dialogue | hooded-claw | Belief: "Penelope is naive and trusts too easily" | "Penelope just announced she found a clue. You're alone in the room. What do you think and say?" | "Does the response reference or build on the belief that Penelope is naive or easily manipulated?" |
| 4 | norm-constraint | hooded-claw | Norm: "Never help Penelope directly" | "Penelope is struggling to reach a high shelf and asks you directly for help." | "Does the response refuse or avoid directly helping Penelope, consistent with the norm constraint?" |

### Observation Construction

Each scenario uses the character's actual `SocialConfig` from `ManorSocialConfigLoader` and their full system prompt from `SystemPromptRenderer`. The observation is built by `CharacterCognition.renderCognitiveSections()` with scenario-appropriate nearby characters and inventory.

The user prompt combines the rendered observation sections with the situation prompt:

```
[Observation sections from CharacterCognition]

SITUATION: [scenario prompt]

Respond in character as JSON with thinking, dialogue, and action fields.
```

### Judge Prompt

The judge is a second LLM invocation using the same `AgentInvocationService`. The judge prompt:

```
You are evaluating whether an AI character's response reflects a specific cognitive state.

COGNITIVE ELEMENT: [the drive/belief/norm being tested]
CHARACTER RESPONSE:
- Thinking: [thinking field]
- Dialogue: [dialogue field]
- Action: [action description]

EVALUATION CRITERIA: [judgeCriteria from scenario]

Score the response 0-5:
0 = No evidence of the cognitive element influencing behavior
1 = Weak/ambiguous evidence
2 = Some evidence but inconsistent
3 = Clear evidence in at least one response field
4 = Strong evidence across multiple fields
5 = The cognitive element clearly drives the entire response

Respond with JSON only: {"score": N, "reasoning": "one sentence"}
```

### Pass Criteria

Each scenario must score **>= 3** (clear evidence in at least one field). This is a generous threshold — the test verifies *influence*, not *dominance*. A score of 3 means the cognitive element is detectable.

The test runs 1 attempt per scenario (no retry on the agent side). The judge has retry logic (3 attempts with backoff) inherited from the existing pattern.

### Test Profile

Uses `@TestProfile` with config override for the scenario mode. Inherits `@Tag("llm-eval")` which is excluded from standard `mvn test` and included only with `-Pllm-eval`.

---

## What's NOT in Scope

- Multi-turn conversation evaluation (existing BaselineLayerTest covers this)
- A/B testing (with/without cognitive sections)
- New judge class — inline judge prompt in the test
- Statistical significance across multiple runs — single run with score threshold

---

## Design Decisions

- D1: Test weight — single-turn + inline LLM judge
- D2: Judge approach — inline LLM judge prompt, no new class
- D3: Test structure — parameterized test with CognitiveScenario records

Full decision records: `specs/issue-59-llm-eval-cognitive/decisions.md`

---

## References

- PromptQualityTest.java — existing eval framework pattern
- BaselineLayerTest.java — existing scenario runner pattern
- EvalJudgeProducer.java — CDI judge wiring
- AgentInvocationService.java:36-44 — LLM invocation interface
- AgentResponse.java — response record (thinking, dialogue, action)
- CharacterCognition.java:94-128 — renderCognitiveSections
- ManorSocialConfigLoader.java — SocialConfig loading
- social-config.yaml — character drives, norms, beliefs
- GE-20260804-2cd3da — split-model evaluation technique
- GE-20260617-f3ea4e — Claude self-judge factual fidelity bias
