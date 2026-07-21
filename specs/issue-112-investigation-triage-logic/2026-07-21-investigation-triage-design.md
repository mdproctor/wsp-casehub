# Investigation Triage Logic — Design Spec

**Issue:** casehubio/aml#112
**Date:** 2026-07-21
**Status:** Approved

## Summary

Replace the `InvestigationTriageWorker` stub (always returns `SAR_WARRANTED`) with
deterministic rule-based triage logic that evaluates specialist findings and decides
whether a SAR is warranted, the transaction is a false positive, or evidence is
inconclusive. Retrofit all AML workers to use typed domain record inputs instead of
raw `Map<String, Object>` extraction.

## Decisions

| Decision | Choice | Rationale |
|----------|--------|-----------|
| Rule structure | Hard-rule gates + scoring fallback | Sanctions/PEP have regulatory implications — must not be bypassable by score arithmetic |
| Worker input typing | Typed domain records via ObjectMapper | Map extraction is fragile (silent typos, wrong casts, no IDE navigation) |
| Threshold configuration | Platform preferences (`PreferenceProvider` + `PreferenceKey`) | Existing platform mechanism; avoids inventing a config approach |
| CBR integration | Threshold adjustment | CBR nudges thresholds based on historical outcomes; hard gates remain absolute |
| Triage output | Structured explanation | Triage is the most consequential decision — must be reconstructable from output alone |
| Evaluator structure | Strategy decomposition (gates + scorer + adjuster) | Each component has one concern; CBR adjustment is isolated and independently testable |

## Domain Model (api/)

### New types

**`TriageInput`** — typed aggregate of the four specialist findings:

```java
public record TriageInput(
    EntityResolutionResult entityResolution,
    PatternAnalysisResult patternAnalysis,
    OsintResult osintScreening,
    CbrPathAdvice cbrPathAdvice  // nullable
) {}
```

**`CbrPathAdvice`** — typed record for CBR path advisor output:

```java
public record CbrPathAdvice(
    int caseCount,
    double avgSimilarity,
    double confidence,
    String predominantOutcome,          // nullable
    Double predominantOutcomeFrequency, // nullable
    boolean error                       // true when advisor failed
) {}
```

**`TriageResult`** — structured triage output:

```java
public record TriageResult(
    TriageDecision decision,
    String reason,
    double riskScore,
    HardGate hardGate,                  // nullable — only when a gate fired
    Double cbrThresholdAdjustment,      // nullable — only when CBR influenced
    List<RiskFactor> factors
) {}
```

**`HardGate`** — enum of absolute gates:

```java
public enum HardGate {
    SANCTIONS_HIT,
    PEP_WITH_SANCTIONS,
    SHELL_COMPANY
}
```

**`RiskFactor`** — individual contributing factor:

```java
public record RiskFactor(String name, double weight, String detail) {}
```

### Modified existing types

**`OsintResult`** — needs `declined` and `reason` fields to match what OSINT workers
actually produce. Current record `OsintResult(sanctionsHit, pepHit, detail)` silently
drops `declined` on deserialization, but triage needs it for uncertainty scoring.

```java
public record OsintResult(
    boolean sanctionsHit,
    boolean pepHit,
    boolean declined,    // added — true when agent lacked clearance
    String reason        // added — replaces 'detail' to match worker output
) {}
```

This is a breaking change to `OsintResult`. All existing callers (buildSummary in
`AmlInvestigationCaseDescriptor`, `InvestigationSummary`, and tests) must be updated.

### Existing types (unchanged)

- `TriageDecision` — `SAR_WARRANTED`, `FALSE_POSITIVE`, `INCONCLUSIVE`
- `EntityResolutionResult` — `entityId`, `ownershipChain`, `entityType`, `riskScore`
- `PatternAnalysisResult` — `structuringDetected`, `description`
- `SuspiciousTransaction` — `id`, `originAccountId`, `destinationAccountId`, `amount`, `currency`, `timestamp`, `flagReason`

## Evaluator Components (api/)

All components are plain classes in `api/` — no CDI, no framework. Testable with
JUnit 5 alone.

### HardGateEvaluator

Checks absolute gates in priority order. Returns `Optional<TriageResult>`.

| Priority | Gate | Condition | Decision |
|----------|------|-----------|----------|
| 1 | `SANCTIONS_HIT` | `osintScreening.sanctionsHit() == true` | SAR_WARRANTED |
| 2 | `PEP_WITH_SANCTIONS` | `osintScreening.pepHit() == true && entityType == "PEP"` | SAR_WARRANTED |
| 3 | `SHELL_COMPANY` | `entityType == "SHELL_COMPANY"` | SAR_WARRANTED |

When a gate fires: `riskScore = 1.0`, the gate enum is set, `cbrThresholdAdjustment = null`,
factors list is empty (gate is the sole reason).

### RiskScorer

Accumulates a weighted risk score from specialist findings. Returns
`ScoringResult(double score, List<RiskFactor> factors)`.

| Factor | Weight | Value |
|--------|--------|-------|
| Entity risk score | 0.35 | `entityResolution.riskScore()` directly |
| Structuring detected | 0.25 | 1.0 if true, 0.0 if false |
| PEP entity type | 0.20 | 1.0 if entityType is PEP, 0.0 otherwise |
| OSINT declined | 0.10 | 0.5 if declined (uncertainty), 0.0 otherwise |
| PEP hit (OSINT) | 0.10 | 1.0 if pepHit, 0.0 otherwise |

Score formula: `Σ(factor_value × weight)` — always in [0.0, 1.0].

OSINT declined gets a partial value (0.5) rather than full — it signals incomplete
information, not confirmed risk.

### CbrAdjuster

Shifts SAR and FALSE_POSITIVE thresholds based on CBR path advice.

- No adjustment when: `cbrPathAdvice` is null, `caseCount == 0`, `confidence < cbrMinConfidence` (default 0.3), or `error == true`
- Predominant `FALSE_POSITIVE` → raises SAR threshold (harder to file SAR)
- Predominant `SAR_WARRANTED` → lowers SAR threshold (easier to file SAR)
- Adjustment magnitude: `confidence × predominantOutcomeFrequency × maxAdjustment`
- Capped at `±maxCbrAdjustment` (default 0.15)

### InvestigationTriageEvaluator

Composes the three components:

```
1. hardGateEvaluator.evaluate(input) → if present, return immediately
2. riskScorer.score(input) → ScoringResult
3. cbrAdjuster.adjust(sarThreshold, fpThreshold, input.cbrPathAdvice())
4. Apply thresholds:
   - score >= adjustedSarThreshold → SAR_WARRANTED
   - score <= adjustedFpThreshold → FALSE_POSITIVE
   - otherwise → INCONCLUSIVE
```

Default thresholds (from preferences):
- SAR threshold: 0.6
- FALSE_POSITIVE threshold: 0.25

## Triage Preferences

**`AmlTriagePolicyKeys`** — follows the `AmlMemoryPolicyKeys` pattern:

| Key | Namespace | Default |
|-----|-----------|---------|
| `SAR_THRESHOLD` | `casehubio.aml.triage` | 0.6 |
| `FALSE_POSITIVE_THRESHOLD` | `casehubio.aml.triage` | 0.25 |
| `MAX_CBR_ADJUSTMENT` | `casehubio.aml.triage` | 0.15 |
| `CBR_MIN_CONFIDENCE` | `casehubio.aml.triage` | 0.3 |

The evaluator components receive resolved threshold values as constructor
parameters (plain doubles). The worker resolves preferences and passes the values
in. This keeps `api/` free of platform preference imports.

## Worker Retrofit — Typed Inputs

Workers that read specialist findings from their input maps get typed deserialization
at the top of their lambda via `objectMapper.convertValue()`.

| Worker | Reads | Deserialization target |
|--------|-------|-----------------------|
| `entityResolutionWorker` | `transaction` | `SuspiciousTransaction` |
| `sarDraftingWorkerJunior` | `transaction`, `entityResolution`, `osintScreening` | Domain records |
| `sarDraftingWorkerSenior` | `transaction`, `entityResolution`, `osintScreening` | Domain records |
| `complianceReviewOpeningWorker` | `transaction`, `entityResolution`, `osintScreening` | Domain records (partially typed already) |
| `InvestigationTriageWorker` | all four findings | `TriageInput` |

Workers that are pure output stubs (`patternAnalysisWorker`, `osintScreeningWorker`,
`osintScreeningWorkerSenior`, `seniorAnalystWorker`) do not need retrofit — they
construct fixed return maps without reading input.

**`InvestigationTriageWorker.create()` signature:**

```java
public static Worker create(ObjectMapper objectMapper, PreferenceProvider preferenceProvider, CurrentPrincipal principal)
```

**`AmlInvestigationCaseDescriptor`** gains `PreferenceProvider` as a new constructor
parameter, wired from `AmlInvestigationCaseHub`.

## YAML Bindings

No changes required. The triage worker's input/output field names are preserved:
- Input projection: `entityResolution`, `patternAnalysis`, `osintScreening`, `cbrPathAdvice`
- Output projection: `investigationTriage: .` (entire return map)
- SAR-drafting condition: `.investigationTriage.decision == "SAR_WARRANTED"`
- `investigation-cleared` goal: `decision == "FALSE_POSITIVE" or decision == "INCONCLUSIVE"`

The output is richer (additional fields) but backward-compatible.

## Testing Strategy

### Unit tests (api/) — pure domain, no Quarkus

**`HardGateEvaluatorTest`:**
- Sanctions hit → SAR_WARRANTED regardless of other signals
- PEP with sanctions → SAR_WARRANTED
- Shell company → SAR_WARRANTED
- Gate priority: sanctions checked before PEP
- No gate triggered → empty Optional
- Low-risk INDIVIDUAL with clean OSINT → no gate fires

**`RiskScorerTest`:**
- High entity risk + structuring → high aggregate score
- Low risk, clean OSINT, no PEP → low aggregate score
- OSINT declined contributes partial score (0.5 × weight)
- Score always in [0.0, 1.0]
- Factors list contains all contributing signals

**`CbrAdjusterTest`:**
- Null CBR advice → thresholds unchanged
- Low confidence → thresholds unchanged
- Predominant FALSE_POSITIVE → SAR threshold raised
- Predominant SAR_WARRANTED → SAR threshold lowered
- Adjustment capped at max
- CBR with error → thresholds unchanged

**`InvestigationTriageEvaluatorTest`:**
- Hard gate fires → immediate return, scorer not consulted
- Score above SAR threshold → SAR_WARRANTED with factors
- Score below FP threshold → FALSE_POSITIVE
- Score in ambiguous band → INCONCLUSIVE
- CBR shifts borderline INCONCLUSIVE → FALSE_POSITIVE
- CBR shifts borderline INCONCLUSIVE → SAR_WARRANTED
- CBR cannot shift a hard-gate decision

### @QuarkusTest integration tests (app/)

**`InvestigationTriageFlowTest`** — retrofit existing + add scenarios:
- SAR path: high risk → SAR_WARRANTED → SAR drafted → compliance review → `investigation-complete`
- FALSE_POSITIVE path: low risk → FALSE_POSITIVE → `investigation-cleared` (no SAR)
- INCONCLUSIVE path: mixed signals → INCONCLUSIVE → `investigation-cleared`
- All paths drain to completion (protocol PP-20260604-820c35)
- Verify structured output in case context

## File Inventory

### New files

| File | Module | Description |
|------|--------|-------------|
| `TriageInput.java` | api | Typed aggregate of specialist findings |
| `CbrPathAdvice.java` | api | Typed CBR path advisor output |
| `TriageResult.java` | api | Structured triage output |
| `HardGate.java` | api | Hard gate enum |
| `RiskFactor.java` | api | Contributing factor record |
| `HardGateEvaluator.java` | api | Absolute gate checker |
| `RiskScorer.java` | api | Weighted risk score accumulator |
| `CbrAdjuster.java` | api | CBR threshold adjustment |
| `InvestigationTriageEvaluator.java` | api | Composer of the three components |
| `AmlTriagePolicyKeys.java` | app | Preference key constants |
| `HardGateEvaluatorTest.java` | api (test) | Unit tests |
| `RiskScorerTest.java` | api (test) | Unit tests |
| `CbrAdjusterTest.java` | api (test) | Unit tests |
| `InvestigationTriageEvaluatorTest.java` | api (test) | Unit tests |
| `TriageInputTest.java` | api (test) | Unit tests |

### Modified files

| File | Change |
|------|--------|
| `InvestigationTriageWorker.java` | Replace stub with typed deserialization + evaluator call |
| `AmlInvestigationCaseDescriptor.java` | Add PreferenceProvider param; retrofit typed inputs in sarDrafting, complianceReview, entityResolution workers |
| `AmlInvestigationCaseHub.java` | Wire PreferenceProvider to descriptor |
| `InvestigationTriageFlowTest.java` | Add FALSE_POSITIVE and INCONCLUSIVE test paths |
| `OsintResult.java` | Add `declined` and `reason` fields; remove `detail` |
| `InvestigationSummary.java` | Update for new `OsintResult` shape (if it constructs OsintResult) |
| `AmlInvestigationCaseDescriptorTest.java` | Update constructor call for new param |
