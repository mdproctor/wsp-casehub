# Personality Calibration Gap — Design Spec

**Date:** 2026-08-04
**Issues:** #12, #13, #14 (epic #11)
**Branch:** issue-12-personality-calibration
**Status:** Draft

## Problem

The 4-layer personality experiment (BASELINE/JUNGIAN/BELBIN/COMPOSITE) established that:
- The disposition encoding pipeline works (7-8/10 MBTI alignment)
- Composition adds measurable behavioral signal (improved TAA for feeling/sensing)
- Rich briefing text overwhelms disposition for thinking/intuition functions

This is a calibration problem, not an architecture problem. Three issues address it:
- **#12** — Catch briefing/disposition conflicts before experiments run
- **#13** — Isolate the framework's independent contribution by varying briefing richness
- **#14** — Strengthen the disposition signal so rich briefings can't overwhelm it

Implementation order: #12 → #13 → #14.

---

## #12 — BriefingCoherenceJudge

### Purpose

Pre-flight validation that detects contradictions between a character's briefing text and their disposition profile. Catches the entire class of problem the 4-layer experiment surfaced (e.g., Peter Perfect's ENFJ expecting Judging behavior while the briefing reads as Perceiving).

### Location

`BriefingCoherenceJudge` in `casehub-eidos-eval`, following the `MbtiAlignmentJudge` / `FunctionActivationJudge` pattern.

### Interface

```java
@ApplicationScoped
public class BriefingCoherenceJudge {

    public CoherenceResult evaluate(String briefingText,
                                     String dispositionDescription,
                                     String agentId) { ... }
}
```

### Input

| Parameter | Source |
|---|---|
| `briefingText` | Raw briefing from the descriptor YAML |
| `dispositionDescription` | Rendered disposition section — MBTI type, function stack, or profile weights depending on the layer |
| `agentId` | For labeling in results |

### Output

```java
record FunctionCoherence(String function,
                         double coherence,
                         String briefingSignal,
                         String dispositionExpectation) {}

record Tension(String function,
               String briefingPhrase,
               String dispositionConflict,
               Severity severity) {}

enum Severity { LOW, MEDIUM, HIGH }

record CoherenceResult(String agentId,
                       List<FunctionCoherence> functions,
                       List<Tension> tensions,
                       double overallCoherence) {}
```

- `FunctionCoherence` — per-function score (0.0–1.0) with what the briefing signals vs what the disposition expects
- `Tension` — specific textual contradiction with severity
- `overallCoherence` — mean of per-function coherence scores

### LLM Judge Prompt

Single LLM call. System prompt instructs the judge to:

1. Read the disposition profile and infer what each of the 8 Jungian functions (Ti, Te, Fi, Fe, Si, Se, Ni, Ne) should express for this character
2. Read the briefing text and identify which functions the briefing language activates
3. Score coherence per function (0.0–1.0): does the briefing reinforce or contradict what the disposition expects?
4. List specific tensions where briefing language works against the disposition, with severity (LOW = minor stylistic mismatch, MEDIUM = mixed signals, HIGH = direct contradiction)

Return as structured JSON matching the `CoherenceResult` shape.

### Integration in wacky-manor

- Add `BriefingCoherenceJudge` to `EvalJudgeProducer`
- Add coherence evaluation pass to `PromptQualityTest` for JUNGIAN and COMPOSITE layers (layers with Jungian disposition data)
- Results go into `prompt-quality.json` under a `"coherence"` key per character

### Applicability

The judge requires Jungian disposition data (MBTI type or function stack) to score against. It is meaningful for JUNGIAN and COMPOSITE layers only. For BASELINE and BELBIN layers (which use simple axes like socialOrient/conflictMode without Jungian mapping), the judge should not be invoked — the `dispositionDescription` wouldn't contain function-level information to score against.

### Success Criteria

- Peter Perfect's ENFJ descriptor (if briefing still has P-reading language) produces a HIGH tension on the J/P dimension
- Characters with well-aligned briefings score > 0.7 overall coherence
- Running coherence check adds < 30s per character (single LLM call)

---

## #13 — Minimal Briefing Experiment

### Purpose

Isolate the disposition framework's independent contribution by systematically varying briefing richness while holding disposition encoding constant. Determines whether the framework adds signal that rich briefings overwhelm, or whether the framework has no independent effect.

### Briefing Richness Levels

| BriefingMode | Content | Example (Hooded Claw) |
|---|---|---|
| `EMPTY` | No briefing | *(briefing field blank)* |
| `NAME_ONLY` | Just the character name | "You are an agent named The Hooded Claw." |
| `NAME_ROLE` | Name + functional role | "You are The Hooded Claw, a villain and secret nemesis." |
| `RICH` | Full existing briefing | *(current descriptors, already baselined)* |

### Implementation

**New enum:** `BriefingMode { EMPTY, NAME_ONLY, NAME_ROLE, RICH }` in `io.casehub.examples.manor.model`.

**Programmatic briefing transform** rather than 12 new YAML files. A utility method takes an `AgentDescriptor` and a `BriefingMode`, returns a copy with the briefing replaced:

```java
class BriefingTransform {
    static AgentDescriptor withBriefing(AgentDescriptor desc, BriefingMode mode) {
        String briefing = switch (mode) {
            case EMPTY -> "";
            case NAME_ONLY -> "You are an agent named " + desc.name() + ".";
            case NAME_ROLE -> "You are " + desc.name() + ", " + inferRole(desc) + ".";
            case RICH -> desc.briefing();
        };
        return desc.withBriefing(briefing); // copy constructor or builder
    }
}
```

If `AgentDescriptor` lacks a `withBriefing` method, construct a new record instance passing all existing fields with the briefing replaced.

**Role inference for NAME_ROLE:** Derived from the descriptor's goals and constraints. A simple map per character:

| Character | Role phrase |
|---|---|
| hooded-claw | "a villain and secret nemesis" |
| penelope-pitstop | "a resourceful Southern belle" |
| ant-hill-mob | "a gang of protective bodyguards" |
| dick-dastardly | "a scheming cheat" |
| peter-perfect | "a gallant hero" |

Hardcoded for the 5 experiment characters — this is an experiment utility, not a general-purpose tool.

### Experiment Matrix

```
for each ProfileMode (BASELINE, JUNGIAN, BELBIN, COMPOSITE):
  for each BriefingMode (EMPTY, NAME_ONLY, NAME_ROLE, RICH):
    for each character (5):
      render system prompt
      evaluate MBTI alignment (JUNGIAN + COMPOSITE only)
      evaluate function activation TAA
```

Total new runs: 3 new briefing modes × 4 layers × 5 characters = 60 evaluation runs.
RICH × all layers = existing baseline (already has data).

### Output

Results keyed as `{layer}-{briefingMode}` in `prompt-quality.json`:
- `"jungian-empty"`, `"jungian-name_only"`, `"jungian-name_role"`, `"jungian-rich"`
- Same per-character structure: MBTI alignment + function activation TAA

### Interpretation Guide

| Observation | Conclusion | Action |
|---|---|---|
| TAA improves as briefing decreases | Briefing overwhelms disposition | Calibration: tune briefing richness or use #14 mechanisms |
| TAA stays flat across briefing levels | Disposition encoding too weak | Architecture: stronger encoding needed in eidos |
| Specific functions improve, others don't | Function-specific briefing interference | Targeted #14 integration for affected functions |
| EMPTY TAA near random (0.5) for BASELINE but high for COMPOSITE | Framework adds real signal when not overwhelmed | Validates the composition approach |

---

## #14 — Stronger Disposition Integration Mechanisms

### Purpose

Three independent mechanisms that make the disposition signal harder for rich briefing text to overwhelm. Each is measured independently via function activation TAA. Blocked by #13 — results from the minimal briefing experiment guide which mechanisms to prioritize.

### Mechanism 1 — Response Format Constraints

Dominant function dictates expected response structure. Added as a `## Response Format` section in the rendered system prompt.

| Dominant Function | Format Constraint |
|---|---|
| Te | "Structure your responses as numbered action plans with explicit criteria for success" |
| Ti | "Present your reasoning as a logical chain: premise → analysis → conclusion" |
| Fe | "Frame your responses around how actions affect the group — who benefits, who is harmed, what's the relational impact" |
| Fi | "Ground your responses in your core values — state what matters to you and why before deciding" |
| Se | "Focus on what's immediately actionable — concrete objects, present dangers, physical options" |
| Si | "Reference what you've seen before, established procedures, and proven approaches" |
| Ni | "Converge to a single strategic insight — one prediction, one pattern, one conclusion" |
| Ne | "Explore multiple possibilities before converging — what-ifs, alternatives, connections" |

**Implementation:** `FunctionFormatConstraint` utility that takes a disposition profile and returns the format instruction for the dominant function. Injected into the system prompt rendering pipeline — either as an eidos template or a post-render transform in wacky-manor.

### Mechanism 2 — Function-Specific Observation Directives

Directives added to the observation context (not the system prompt) that remind the character how to process what they see. Added to `ObservationBuilder.buildObservation()` as a `== Cognitive Approach ==` section.

Example for Te-dominant Hooded Claw:
```
== Cognitive Approach ==
As a systematic strategic thinker, prioritise structured execution over
creative exploration. Evaluate each option against your objectives.
Organise your approach into clear steps.
```

**Implementation:** `ObservationBuilder` reads the character's disposition profile (from the descriptor, available via `CharacterState`) and appends a cognitive approach directive. This is wacky-manor local — the observation builder is project code, not eidos.

### Mechanism 3 — Schema Reinforcement

Add a function-aligned `reasoning` field to the `AgentResponse` JSON schema. The field description varies by dominant function, steering the LLM's reasoning process.

Example for Te-dominant:
```json
{
  "reasoning": "(describe your systematic analysis — what options exist, which is optimal, why)",
  "thinking": "...",
  "dialogue": "...",
  "action": {...}
}
```

**Implementation:** The structured output prompt describes the JSON schema in the system prompt. Parameterise the `reasoning` field description based on the character's disposition profile.

### Experiment Design

Each mechanism gets a configuration flag. The experiment tests each independently and all combined:

```
for each variant (FORMAT_CONSTRAINT, OBSERVATION_DIRECTIVE, SCHEMA_REINFORCEMENT, ALL_THREE):
  for ProfileMode COMPOSITE (richest disposition data):
    for BriefingMode RICH (the problem case):
      for each character (5):
        evaluate function activation TAA
```

4 variants × 5 characters × 2 scenarios = 40 evaluation runs.
Compare TAA against COMPOSITE/RICH baseline from #13.

### What We Learn

- Which mechanism moves TAA the most
- Whether they're additive (ALL_THREE > any individual)
- Whether different functions respond to different mechanisms (e.g., format constraints help Te/Ti but observation directives help Fe/Fi)

---

## Cross-Cutting Concerns

### EvalFilter Extension

The existing `EvalFilter` supports `eval.characters` and `eval.layers` system properties. Extend with `eval.briefings` for BriefingMode filtering, allowing incremental experiment runs:

```bash
mvn test -pl wacky-manor -Pllm-eval -Deval.layers=composite -Deval.briefings=empty,name_only
```

### Result Accumulation

`PromptQualityTest` already accumulates results incrementally into `prompt-quality.json`. The new matrix dimensions (briefing mode, integration mechanism) extend the key structure without breaking existing results.

### AgentDescriptor Copy

`AgentDescriptor` is a record in `casehub-eidos-api`. Creating a copy with a modified briefing requires either:
1. A `withBriefing(String)` method added to eidos-api (cleanest)
2. Constructing a new record with all fields copied (verbose but no cross-repo change)

Prefer option 1 if we're already committing to eidos-eval for the CoherenceJudge.

### Cost Estimate

| Issue | LLM Calls | Estimated Time |
|---|---|---|
| #12 coherence (5 chars × 2 layers) | 10 | ~2 min |
| #13 full matrix (60 new runs × 3 calls each) | 180 | ~30 min |
| #14 mechanism evaluation (40 runs × 2 calls each) | 80 | ~15 min |
| **Total** | **270** | **~47 min** |

All runs are via the `llm-eval` Maven profile — excluded from CI, developer-triggered only.
