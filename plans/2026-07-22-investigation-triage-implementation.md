# Investigation Triage Logic — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #112 — feat: investigation triage logic — replace SAR_WARRANTED stub
**Issue group:** #112

**Goal:** Replace the InvestigationTriageWorker stub with deterministic rule-based
triage logic (hard gates + scoring + CBR threshold adjustment) and retrofit all AML
workers to typed domain record inputs.

**Architecture:** Three evaluator components in `api/` (HardGateEvaluator, RiskScorer,
CbrAdjuster) composed by InvestigationTriageEvaluator. The triage worker in `app/`
resolves platform preferences, constructs the evaluator, and returns a structured
TriageResult. INCONCLUSIVE outcomes declare a PlannedAction(INVESTIGATION_CLEARANCE)
for human oversight. Worker retrofit replaces raw Map extraction with ObjectMapper
deserialization to typed domain records.

**Tech Stack:** Java 21, Quarkus 3.32.2, JUnit 5, AssertJ, REST Assured, Awaitility

## Global Constraints

- All evaluator components live in `api/` — pure domain, no CDI, no Quarkus
- Worker code lives in `app/` — CDI wiring, ObjectMapper, PreferenceProvider
- FlowWorkerFunction does not support PlannedAction (engine#564) — triage uses WorkerFunction.Sync
- Platform preferences via `PreferenceKey<DoublePreference>` from `io.casehub.api.spi.routing`
- YAML bindings unchanged — output shape is backward-compatible (additive fields)
- OsintResult `detail` → `reason` rename: no existing code calls `.detail()` on OsintResult
- `EntityResolutionResult` riskScore values in existing code are all valid (0.35, 0.87, 0.92)
- Project root: `/Users/mdproctor/claude/casehub/worktrees/27/aml`
- IntelliJ project: `/Users/mdproctor/claude/casehub/aml` (navigation only — worktree has no .idea)
- Build/test: `mvn -pl <module> -am test -Dtest=ClassName -Dsurefire.failIfNoSpecifiedTests=false`

---

### Task 1: OsintResult and EntityResolutionResult breaking changes

Breaking changes to existing domain records. Must be done first — every
subsequent task depends on the updated types.

**Files:**
- Modify: `api/src/main/java/io/casehub/aml/domain/OsintResult.java`
- Modify: `api/src/main/java/io/casehub/aml/domain/EntityResolutionResult.java`
- Modify: `app/src/main/java/io/casehub/aml/DefaultOsintScreeningService.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java` (buildSummary)
- Modify: `app/src/test/java/io/casehub/aml/AmlInvestigationCoordinatorTest.java` (2 sites)
- Modify: `app/src/test/java/io/casehub/aml/DefaultSarDraftingServiceTest.java` (1 site)
- Test: `api/src/test/java/io/casehub/aml/domain/OsintResultTest.java` (new)
- Test: `api/src/test/java/io/casehub/aml/domain/EntityResolutionResultTest.java` (new)

**Interfaces:**
- Produces: `OsintResult(boolean sanctionsHit, boolean pepHit, boolean declined, String reason)` with compact constructor validation
- Produces: `EntityResolutionResult(String entityId, String ownershipChain, String entityType, double riskScore)` with riskScore [0.0, 1.0] validation

- [ ] **Step 1: Write OsintResult validation tests**

Create `api/src/test/java/io/casehub/aml/domain/OsintResultTest.java`:

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class OsintResultTest {

    @Test
    void completedScreening_sanctionsHit() {
        var result = new OsintResult(true, false, false, "OFAC/SDN match");
        assertThat(result.sanctionsHit()).isTrue();
        assertThat(result.declined()).isFalse();
    }

    @Test
    void completedScreening_pepHit() {
        var result = new OsintResult(false, true, false, "PEP database match");
        assertThat(result.pepHit()).isTrue();
    }

    @Test
    void completedScreening_clean() {
        var result = new OsintResult(false, false, false, "no matches");
        assertThat(result.sanctionsHit()).isFalse();
        assertThat(result.pepHit()).isFalse();
        assertThat(result.declined()).isFalse();
    }

    @Test
    void declinedScreening_valid() {
        var result = new OsintResult(false, false, true, "insufficient clearance");
        assertThat(result.declined()).isTrue();
        assertThat(result.reason()).isEqualTo("insufficient clearance");
    }

    @Test
    void declined_withSanctionsHit_throws() {
        assertThatThrownBy(() -> new OsintResult(true, false, true, "declined"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("declined screening cannot report sanctions or PEP hits");
    }

    @Test
    void declined_withPepHit_throws() {
        assertThatThrownBy(() -> new OsintResult(false, true, true, "declined"))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("declined screening cannot report sanctions or PEP hits");
    }
}
```

- [ ] **Step 2: Write EntityResolutionResult validation tests**

Create `api/src/test/java/io/casehub/aml/domain/EntityResolutionResultTest.java`:

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class EntityResolutionResultTest {

    @Test
    void validRiskScore() {
        var result = new EntityResolutionResult("E-1", "chain", "CORPORATE", 0.5);
        assertThat(result.riskScore()).isEqualTo(0.5);
    }

    @Test
    void riskScoreAtBounds() {
        assertThat(new EntityResolutionResult("E-1", "c", "PEP", 0.0).riskScore()).isEqualTo(0.0);
        assertThat(new EntityResolutionResult("E-1", "c", "PEP", 1.0).riskScore()).isEqualTo(1.0);
    }

    @Test
    void riskScoreBelowZero_throws() {
        assertThatThrownBy(() -> new EntityResolutionResult("E-1", "c", "PEP", -0.1))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("riskScore must be in [0.0, 1.0]");
    }

    @Test
    void riskScoreAboveOne_throws() {
        assertThatThrownBy(() -> new EntityResolutionResult("E-1", "c", "PEP", 1.1))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("riskScore must be in [0.0, 1.0]");
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn -pl api -am test -Dtest="OsintResultTest,EntityResolutionResultTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure (OsintResult has 3 params, tests use 4; EntityResolutionResult has no compact constructor)

- [ ] **Step 4: Implement OsintResult changes**

Replace `api/src/main/java/io/casehub/aml/domain/OsintResult.java`:

```java
package io.casehub.aml.domain;

public record OsintResult(
        boolean sanctionsHit,
        boolean pepHit,
        boolean declined,
        String reason) {
    public OsintResult {
        if (declined && (sanctionsHit || pepHit)) {
            throw new IllegalArgumentException(
                "declined screening cannot report sanctions or PEP hits");
        }
    }
}
```

- [ ] **Step 5: Implement EntityResolutionResult validation**

Replace `api/src/main/java/io/casehub/aml/domain/EntityResolutionResult.java`:

```java
package io.casehub.aml.domain;

public record EntityResolutionResult(
        String entityId,
        String ownershipChain,
        String entityType,
        double riskScore) {
    public EntityResolutionResult {
        if (riskScore < 0.0 || riskScore > 1.0) {
            throw new IllegalArgumentException(
                "riskScore must be in [0.0, 1.0], got: " + riskScore);
        }
    }
}
```

- [ ] **Step 6: Fix all OsintResult constructor call sites**

Update every `new OsintResult(a, b, c)` → `new OsintResult(a, b, false, c)`:

| File | Line | Old | New |
|------|------|-----|-----|
| `DefaultOsintScreeningService.java` | 11 | `new OsintResult(false, false, "")` | `new OsintResult(false, false, false, "")` |
| `AmlInvestigationCaseDescriptor.java` | 290 | `new OsintResult(false, false, "no matches")` | `new OsintResult(false, false, false, "no matches")` |
| `AmlInvestigationCoordinatorTest.java` | 48 | `new OsintResult(false, false, "clean")` | `new OsintResult(false, false, false, "clean")` |
| `AmlInvestigationCoordinatorTest.java` | 77 | `new OsintResult(false, false, "clean")` | `new OsintResult(false, false, false, "clean")` |
| `DefaultSarDraftingServiceTest.java` | 31 | `new OsintResult(false, false, "clean")` | `new OsintResult(false, false, false, "clean")` |

- [ ] **Step 7: Run all tests to verify green**

Run: `mvn -pl api -am test -Dtest="OsintResultTest,EntityResolutionResultTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS (all existing tests still green with updated constructors)

- [ ] **Step 8: Commit**

```
feat(#112): OsintResult — add declined/reason fields; EntityResolutionResult — riskScore validation

Refs #112
```

---

### Task 2: Triage domain types

Pure data types with no logic. Foundation for the evaluator components.

**Files:**
- Create: `api/src/main/java/io/casehub/aml/domain/CbrPathAdvice.java`
- Create: `api/src/main/java/io/casehub/aml/domain/HardGate.java`
- Create: `api/src/main/java/io/casehub/aml/domain/RiskFactor.java`
- Create: `api/src/main/java/io/casehub/aml/domain/TriageInput.java`
- Create: `api/src/main/java/io/casehub/aml/domain/TriageResult.java`
- Test: `api/src/test/java/io/casehub/aml/domain/TriageInputTest.java` (new)

**Interfaces:**
- Produces: `TriageInput(EntityResolutionResult, PatternAnalysisResult, OsintResult, CbrPathAdvice)`
- Produces: `CbrPathAdvice(int caseCount, double avgSimilarity, double confidence, String predominantOutcome, Double predominantOutcomeFrequency, boolean error)`
- Produces: `TriageResult(TriageDecision, String reason, double riskScore, HardGate, Double cbrThresholdAdjustment, List<RiskFactor>)`
- Produces: `HardGate` enum: `SANCTIONS_HIT`, `CONFIRMED_PEP`, `SHELL_COMPANY`
- Produces: `RiskFactor(String name, double weight, String detail)`

- [ ] **Step 1: Write TriageInput construction test**

Create `api/src/test/java/io/casehub/aml/domain/TriageInputTest.java`:

```java
package io.casehub.aml.domain;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class TriageInputTest {

    @Test
    void constructionWithNullCbrAdvice() {
        var input = new TriageInput(
                new EntityResolutionResult("E-1", "chain", "CORPORATE", 0.35),
                new PatternAnalysisResult(false, "no pattern"),
                new OsintResult(false, false, false, "clean"),
                null);
        assertThat(input.cbrPathAdvice()).isNull();
        assertThat(input.entityResolution().entityType()).isEqualTo("CORPORATE");
    }

    @Test
    void constructionWithCbrAdvice() {
        var cbr = new CbrPathAdvice(5, 0.7, 0.6, "SAR_WARRANTED", 0.8, false);
        var input = new TriageInput(
                new EntityResolutionResult("E-1", "chain", "PEP", 0.87),
                new PatternAnalysisResult(true, "structuring"),
                new OsintResult(false, false, false, "clean"),
                cbr);
        assertThat(input.cbrPathAdvice().predominantOutcome()).isEqualTo("SAR_WARRANTED");
    }

    @Test
    void cbrPathAdvice_errorCase() {
        var cbr = new CbrPathAdvice(0, 0.0, 0.0, null, null, true);
        assertThat(cbr.error()).isTrue();
        assertThat(cbr.predominantOutcome()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl api -am test -Dtest=TriageInputTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure (types don't exist)

- [ ] **Step 3: Create all domain types**

Create `api/src/main/java/io/casehub/aml/domain/HardGate.java`:
```java
package io.casehub.aml.domain;

public enum HardGate {
    SANCTIONS_HIT,
    CONFIRMED_PEP,
    SHELL_COMPANY
}
```

Create `api/src/main/java/io/casehub/aml/domain/RiskFactor.java`:
```java
package io.casehub.aml.domain;

public record RiskFactor(String name, double weight, String detail) {}
```

Create `api/src/main/java/io/casehub/aml/domain/CbrPathAdvice.java`:
```java
package io.casehub.aml.domain;

public record CbrPathAdvice(
        int caseCount,
        double avgSimilarity,
        double confidence,
        String predominantOutcome,
        Double predominantOutcomeFrequency,
        boolean error) {}
```

Create `api/src/main/java/io/casehub/aml/domain/TriageResult.java`:
```java
package io.casehub.aml.domain;

import java.util.List;

public record TriageResult(
        TriageDecision decision,
        String reason,
        double riskScore,
        HardGate hardGate,
        Double cbrThresholdAdjustment,
        List<RiskFactor> factors) {}
```

Create `api/src/main/java/io/casehub/aml/domain/TriageInput.java`:
```java
package io.casehub.aml.domain;

public record TriageInput(
        EntityResolutionResult entityResolution,
        PatternAnalysisResult patternAnalysis,
        OsintResult osintScreening,
        CbrPathAdvice cbrPathAdvice) {}
```

- [ ] **Step 4: Run test to verify green**

Run: `mvn -pl api -am test -Dtest=TriageInputTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#112): triage domain types — TriageInput, CbrPathAdvice, TriageResult, HardGate, RiskFactor

Refs #112
```

---

### Task 3: HardGateEvaluator

First evaluator component. Checks absolute regulatory gates.

**Files:**
- Create: `api/src/main/java/io/casehub/aml/triage/HardGateEvaluator.java`
- Test: `api/src/test/java/io/casehub/aml/triage/HardGateEvaluatorTest.java` (new)

**Interfaces:**
- Consumes: `TriageInput`, `OsintResult`, `EntityResolutionResult`, `TriageResult`, `HardGate`, `RiskFactor`, `TriageDecision`
- Produces: `HardGateEvaluator.evaluate(TriageInput) → Optional<TriageResult>`

- [ ] **Step 1: Write failing tests**

Create `api/src/test/java/io/casehub/aml/triage/HardGateEvaluatorTest.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.*;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class HardGateEvaluatorTest {

    private final HardGateEvaluator evaluator = new HardGateEvaluator();

    private TriageInput input(String entityType, double riskScore,
                              boolean sanctionsHit, boolean pepHit, boolean declined) {
        return new TriageInput(
                new EntityResolutionResult("E-1", "chain", entityType, riskScore),
                new PatternAnalysisResult(false, "none"),
                new OsintResult(sanctionsHit, pepHit, declined, "reason"),
                null);
    }

    @Test
    void sanctionsHit_returnsSarWarranted() {
        var result = evaluator.evaluate(input("CORPORATE", 0.2, true, false, false));
        assertThat(result).isPresent();
        assertThat(result.get().decision()).isEqualTo(TriageDecision.SAR_WARRANTED);
        assertThat(result.get().hardGate()).isEqualTo(HardGate.SANCTIONS_HIT);
        assertThat(result.get().riskScore()).isEqualTo(1.0);
    }

    @Test
    void confirmedPep_returnsSarWarranted() {
        var result = evaluator.evaluate(input("PEP", 0.5, false, true, false));
        assertThat(result).isPresent();
        assertThat(result.get().decision()).isEqualTo(TriageDecision.SAR_WARRANTED);
        assertThat(result.get().hardGate()).isEqualTo(HardGate.CONFIRMED_PEP);
    }

    @Test
    void pepHit_butNotPepEntityType_noGate() {
        var result = evaluator.evaluate(input("CORPORATE", 0.5, false, true, false));
        assertThat(result).isEmpty();
    }

    @Test
    void shellCompany_returnsSarWarranted() {
        var result = evaluator.evaluate(input("SHELL_COMPANY", 0.3, false, false, false));
        assertThat(result).isPresent();
        assertThat(result.get().decision()).isEqualTo(TriageDecision.SAR_WARRANTED);
        assertThat(result.get().hardGate()).isEqualTo(HardGate.SHELL_COMPANY);
    }

    @Test
    void sanctionsHit_takesPriorityOverPep() {
        var result = evaluator.evaluate(input("PEP", 0.9, true, true, false));
        assertThat(result).isPresent();
        assertThat(result.get().hardGate()).isEqualTo(HardGate.SANCTIONS_HIT);
    }

    @Test
    void lowRiskIndividual_cleanOsint_noGate() {
        var result = evaluator.evaluate(input("INDIVIDUAL", 0.1, false, false, false));
        assertThat(result).isEmpty();
    }

    @Test
    void declinedOsint_noGate() {
        var result = evaluator.evaluate(input("CORPORATE", 0.5, false, false, true));
        assertThat(result).isEmpty();
    }

    @Test
    void gateResult_hasEmptyFactorsList() {
        var result = evaluator.evaluate(input("SHELL_COMPANY", 0.3, false, false, false));
        assertThat(result.get().factors()).isEmpty();
        assertThat(result.get().cbrThresholdAdjustment()).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl api -am test -Dtest=HardGateEvaluatorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure

- [ ] **Step 3: Implement HardGateEvaluator**

Create `api/src/main/java/io/casehub/aml/triage/HardGateEvaluator.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.*;

import java.util.List;
import java.util.Optional;

public final class HardGateEvaluator {

    public Optional<TriageResult> evaluate(TriageInput input) {
        var osint = input.osintScreening();
        var entityType = input.entityResolution().entityType();

        if (osint.sanctionsHit()) {
            return gate(HardGate.SANCTIONS_HIT, "Hard gate: sanctions hit on OFAC/SDN screening");
        }
        if (osint.pepHit() && "PEP".equals(entityType)) {
            return gate(HardGate.CONFIRMED_PEP, "Hard gate: confirmed PEP with OSINT PEP hit");
        }
        if ("SHELL_COMPANY".equals(entityType)) {
            return gate(HardGate.SHELL_COMPANY, "Hard gate: shell company entity type");
        }
        return Optional.empty();
    }

    private Optional<TriageResult> gate(HardGate gate, String reason) {
        return Optional.of(new TriageResult(
                TriageDecision.SAR_WARRANTED, reason, 1.0, gate, null, List.of()));
    }
}
```

- [ ] **Step 4: Run tests to verify green**

Run: `mvn -pl api -am test -Dtest=HardGateEvaluatorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#112): HardGateEvaluator — absolute regulatory gates for sanctions, PEP, shell company

Refs #112
```

---

### Task 4: RiskScorer

Weighted risk score accumulator.

**Files:**
- Create: `api/src/main/java/io/casehub/aml/triage/RiskScorer.java`
- Test: `api/src/test/java/io/casehub/aml/triage/RiskScorerTest.java` (new)

**Interfaces:**
- Consumes: `TriageInput`, `RiskFactor`
- Produces: `RiskScorer.score(TriageInput) → RiskScorer.ScoringResult(double score, List<RiskFactor> factors)`

- [ ] **Step 1: Write failing tests**

Create `api/src/test/java/io/casehub/aml/triage/RiskScorerTest.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.*;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class RiskScorerTest {

    private final RiskScorer scorer = new RiskScorer();

    private TriageInput input(String entityType, double riskScore,
                              boolean structuring, boolean pepHit,
                              boolean sanctionsHit, boolean declined) {
        return new TriageInput(
                new EntityResolutionResult("E-1", "chain", entityType, riskScore),
                new PatternAnalysisResult(structuring, "desc"),
                new OsintResult(sanctionsHit, pepHit, declined, "reason"),
                null);
    }

    @Test
    void highRisk_structuring_pep_yieldsHighScore() {
        var result = scorer.score(input("PEP", 0.9, true, true, false, false));
        // 0.9*0.35 + 1.0*0.25 + 1.0*0.20 + 0.0*0.10 + 1.0*0.10 = 0.315 + 0.25 + 0.20 + 0.0 + 0.10 = 0.865
        assertThat(result.score()).isCloseTo(0.865, org.assertj.core.data.Offset.offset(0.001));
    }

    @Test
    void lowRisk_noFlags_yieldsLowScore() {
        var result = scorer.score(input("INDIVIDUAL", 0.1, false, false, false, false));
        // 0.1*0.35 + 0.0*0.25 + 0.0*0.20 + 0.0*0.10 + 0.0*0.10 = 0.035
        assertThat(result.score()).isCloseTo(0.035, org.assertj.core.data.Offset.offset(0.001));
    }

    @Test
    void osintDeclined_contributesPartialScore() {
        var result = scorer.score(input("INDIVIDUAL", 0.1, false, false, false, true));
        // 0.1*0.35 + 0.0*0.25 + 0.0*0.20 + 0.5*0.10 + 0.0*0.10 = 0.035 + 0.05 = 0.085
        assertThat(result.score()).isCloseTo(0.085, org.assertj.core.data.Offset.offset(0.001));
    }

    @Test
    void scoreAlwaysInUnitRange() {
        var maxInput = input("PEP", 1.0, true, true, false, false);
        assertThat(scorer.score(maxInput).score()).isLessThanOrEqualTo(1.0);

        var minInput = input("INDIVIDUAL", 0.0, false, false, false, false);
        assertThat(scorer.score(minInput).score()).isGreaterThanOrEqualTo(0.0);
    }

    @Test
    void factorsListContainsAllSignals() {
        var result = scorer.score(input("PEP", 0.5, true, false, false, false));
        assertThat(result.factors()).hasSizeGreaterThanOrEqualTo(3);
        assertThat(result.factors()).extracting(RiskFactor::name)
                .contains("entity-risk-score", "structuring-detected", "pep-entity-type");
    }

    @Test
    void entityRiskScoreIsClamped() {
        // Even though EntityResolutionResult now validates, defense-in-depth
        var normalInput = input("CORPORATE", 0.5, false, false, false, false);
        var result = scorer.score(normalInput);
        assertThat(result.score()).isGreaterThanOrEqualTo(0.0);
        assertThat(result.score()).isLessThanOrEqualTo(1.0);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl api -am test -Dtest=RiskScorerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure

- [ ] **Step 3: Implement RiskScorer**

Create `api/src/main/java/io/casehub/aml/triage/RiskScorer.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.RiskFactor;
import io.casehub.aml.domain.TriageInput;

import java.util.ArrayList;
import java.util.List;

public final class RiskScorer {

    public record ScoringResult(double score, List<RiskFactor> factors) {}

    public ScoringResult score(TriageInput input) {
        var factors = new ArrayList<RiskFactor>();
        double total = 0.0;

        double entityRisk = Math.max(0.0, Math.min(1.0, input.entityResolution().riskScore()));
        total += entityRisk * 0.35;
        factors.add(new RiskFactor("entity-risk-score", 0.35,
                "riskScore=" + entityRisk));

        double structuring = input.patternAnalysis().structuringDetected() ? 1.0 : 0.0;
        total += structuring * 0.25;
        if (structuring > 0) {
            factors.add(new RiskFactor("structuring-detected", 0.25,
                    input.patternAnalysis().description()));
        }

        boolean isPep = "PEP".equals(input.entityResolution().entityType());
        double pepType = isPep ? 1.0 : 0.0;
        total += pepType * 0.20;
        if (isPep) {
            factors.add(new RiskFactor("pep-entity-type", 0.20,
                    "entityType=PEP"));
        }

        double declinedValue = input.osintScreening().declined() ? 0.5 : 0.0;
        total += declinedValue * 0.10;
        if (input.osintScreening().declined()) {
            factors.add(new RiskFactor("osint-declined", 0.10,
                    "screening declined — uncertainty factor (0.5)"));
        }

        double pepHit = input.osintScreening().pepHit() ? 1.0 : 0.0;
        total += pepHit * 0.10;
        if (input.osintScreening().pepHit()) {
            factors.add(new RiskFactor("osint-pep-hit", 0.10,
                    "PEP database match"));
        }

        return new ScoringResult(total, List.copyOf(factors));
    }
}
```

- [ ] **Step 4: Run tests to verify green**

Run: `mvn -pl api -am test -Dtest=RiskScorerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#112): RiskScorer — weighted risk score accumulator with five specialist signal factors

Refs #112
```

---

### Task 5: CbrAdjuster

CBR threshold adjustment with ordering invariant.

**Files:**
- Create: `api/src/main/java/io/casehub/aml/triage/CbrAdjuster.java`
- Test: `api/src/test/java/io/casehub/aml/triage/CbrAdjusterTest.java` (new)

**Interfaces:**
- Consumes: `CbrPathAdvice`
- Produces: `CbrAdjuster(double maxAdjustment, double minConfidence)`
- Produces: `CbrAdjuster.adjust(double sarThreshold, double fpThreshold, CbrPathAdvice) → CbrAdjuster.AdjustedThresholds(double sarThreshold, double fpThreshold, Double adjustment)`

- [ ] **Step 1: Write failing tests**

Create `api/src/test/java/io/casehub/aml/triage/CbrAdjusterTest.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.CbrPathAdvice;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.data.Offset.offset;

class CbrAdjusterTest {

    private final CbrAdjuster adjuster = new CbrAdjuster(0.15, 0.3);

    @Test
    void nullAdvice_noAdjustment() {
        var result = adjuster.adjust(0.6, 0.25, null);
        assertThat(result.sarThreshold()).isEqualTo(0.6);
        assertThat(result.fpThreshold()).isEqualTo(0.25);
        assertThat(result.adjustment()).isNull();
    }

    @Test
    void zeroCaseCount_noAdjustment() {
        var cbr = new CbrPathAdvice(0, 0.0, 0.0, null, null, false);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        assertThat(result.adjustment()).isNull();
    }

    @Test
    void lowConfidence_noAdjustment() {
        var cbr = new CbrPathAdvice(3, 0.5, 0.2, "SAR_WARRANTED", 0.8, false);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        assertThat(result.adjustment()).isNull();
    }

    @Test
    void error_noAdjustment() {
        var cbr = new CbrPathAdvice(5, 0.7, 0.6, "SAR_WARRANTED", 0.8, true);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        assertThat(result.adjustment()).isNull();
    }

    @Test
    void nullPredominantOutcome_noAdjustment() {
        var cbr = new CbrPathAdvice(5, 0.7, 0.6, null, null, false);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        assertThat(result.adjustment()).isNull();
    }

    @Test
    void predominantFalsePositive_raisesThresholds() {
        var cbr = new CbrPathAdvice(10, 0.8, 0.7, "FALSE_POSITIVE", 0.9, false);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        assertThat(result.sarThreshold()).isGreaterThan(0.6);
        assertThat(result.fpThreshold()).isGreaterThan(0.25);
        assertThat(result.adjustment()).isNotNull();
    }

    @Test
    void predominantSarWarranted_lowersThresholds() {
        var cbr = new CbrPathAdvice(10, 0.8, 0.7, "SAR_WARRANTED", 0.9, false);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        assertThat(result.sarThreshold()).isLessThan(0.6);
        assertThat(result.fpThreshold()).isLessThan(0.25);
        assertThat(result.adjustment()).isNotNull();
    }

    @Test
    void predominantInconclusive_noAdjustment() {
        var cbr = new CbrPathAdvice(10, 0.8, 0.7, "INCONCLUSIVE", 0.9, false);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        assertThat(result.adjustment()).isNull();
    }

    @Test
    void adjustmentCappedAtMax() {
        var cbr = new CbrPathAdvice(20, 0.95, 0.95, "FALSE_POSITIVE", 1.0, false);
        var result = adjuster.adjust(0.6, 0.25, cbr);
        // magnitude = 0.95 * 1.0 * 0.15 = 0.1425 — within cap
        assertThat(result.sarThreshold()).isLessThanOrEqualTo(0.6 + 0.15);
        assertThat(result.fpThreshold()).isLessThanOrEqualTo(0.25 + 0.15);
    }

    @Test
    void thresholdOrderingInvariant_clampedWhenViolated() {
        // Start with thresholds very close, strong CBR should not cross them
        var cbr = new CbrPathAdvice(20, 0.95, 0.95, "FALSE_POSITIVE", 1.0, false);
        var result = adjuster.adjust(0.35, 0.30, cbr);
        // After large upward shift, SAR could cross FP — invariant should prevent
        assertThat(result.sarThreshold()).isGreaterThan(result.fpThreshold());
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl api -am test -Dtest=CbrAdjusterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure

- [ ] **Step 3: Implement CbrAdjuster**

Create `api/src/main/java/io/casehub/aml/triage/CbrAdjuster.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.CbrPathAdvice;

import java.util.logging.Logger;

public final class CbrAdjuster {

    private static final Logger LOG = Logger.getLogger(CbrAdjuster.class.getName());

    private final double maxAdjustment;
    private final double minConfidence;

    public CbrAdjuster(double maxAdjustment, double minConfidence) {
        this.maxAdjustment = maxAdjustment;
        this.minConfidence = minConfidence;
    }

    public record AdjustedThresholds(double sarThreshold, double fpThreshold, Double adjustment) {}

    public AdjustedThresholds adjust(double sarThreshold, double fpThreshold, CbrPathAdvice cbr) {
        if (cbr == null || cbr.caseCount() == 0 || cbr.confidence() < minConfidence
                || cbr.error() || cbr.predominantOutcome() == null) {
            return new AdjustedThresholds(sarThreshold, fpThreshold, null);
        }

        String outcome = cbr.predominantOutcome();
        if ("INCONCLUSIVE".equals(outcome)) {
            return new AdjustedThresholds(sarThreshold, fpThreshold, null);
        }

        double freq = cbr.predominantOutcomeFrequency() != null ? cbr.predominantOutcomeFrequency() : 0.0;
        double magnitude = Math.min(cbr.confidence() * freq * maxAdjustment, maxAdjustment);

        double adjustedSar;
        double adjustedFp;
        if ("FALSE_POSITIVE".equals(outcome)) {
            adjustedSar = sarThreshold + magnitude;
            adjustedFp = fpThreshold + magnitude;
        } else if ("SAR_WARRANTED".equals(outcome)) {
            adjustedSar = sarThreshold - magnitude;
            adjustedFp = fpThreshold - magnitude;
        } else {
            return new AdjustedThresholds(sarThreshold, fpThreshold, null);
        }

        if (adjustedSar <= adjustedFp) {
            double mid = (adjustedSar + adjustedFp) / 2.0;
            adjustedSar = mid + 0.05;
            adjustedFp = mid - 0.05;
            LOG.warning("CBR threshold ordering violated — clamped to midpoint ± 0.05");
        }

        return new AdjustedThresholds(adjustedSar, adjustedFp, magnitude);
    }
}
```

- [ ] **Step 4: Run tests to verify green**

Run: `mvn -pl api -am test -Dtest=CbrAdjusterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#112): CbrAdjuster — CBR threshold adjustment with ordering invariant

Refs #112
```

---

### Task 6: InvestigationTriageEvaluator

Composes all three components.

**Files:**
- Create: `api/src/main/java/io/casehub/aml/triage/InvestigationTriageEvaluator.java`
- Test: `api/src/test/java/io/casehub/aml/triage/InvestigationTriageEvaluatorTest.java` (new)

**Interfaces:**
- Consumes: `HardGateEvaluator`, `RiskScorer`, `CbrAdjuster`, `TriageInput`, `TriageResult`
- Produces: `InvestigationTriageEvaluator(double sarThreshold, double fpThreshold, double maxCbrAdjustment, double cbrMinConfidence)`
- Produces: `InvestigationTriageEvaluator.evaluate(TriageInput) → TriageResult`

- [ ] **Step 1: Write failing tests**

Create `api/src/test/java/io/casehub/aml/triage/InvestigationTriageEvaluatorTest.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.*;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

class InvestigationTriageEvaluatorTest {

    private final InvestigationTriageEvaluator evaluator =
            new InvestigationTriageEvaluator(0.6, 0.25, 0.15, 0.3);

    private TriageInput input(String entityType, double riskScore,
                              boolean structuring, boolean pepHit,
                              boolean sanctionsHit, boolean declined,
                              CbrPathAdvice cbr) {
        return new TriageInput(
                new EntityResolutionResult("E-1", "chain", entityType, riskScore),
                new PatternAnalysisResult(structuring, "desc"),
                new OsintResult(sanctionsHit, pepHit, declined, "reason"),
                cbr);
    }

    @Test
    void hardGate_sanctionsHit_immediateReturn() {
        var result = evaluator.evaluate(
                input("INDIVIDUAL", 0.1, false, false, true, false, null));
        assertThat(result.decision()).isEqualTo(TriageDecision.SAR_WARRANTED);
        assertThat(result.hardGate()).isEqualTo(HardGate.SANCTIONS_HIT);
    }

    @Test
    void highScore_sarWarranted() {
        // PEP with high risk + structuring → score > 0.6
        var result = evaluator.evaluate(
                input("PEP", 0.9, true, false, false, false, null));
        assertThat(result.decision()).isEqualTo(TriageDecision.SAR_WARRANTED);
        assertThat(result.hardGate()).isNull();
        assertThat(result.factors()).isNotEmpty();
    }

    @Test
    void lowScore_falsePositive() {
        var result = evaluator.evaluate(
                input("INDIVIDUAL", 0.1, false, false, false, false, null));
        assertThat(result.decision()).isEqualTo(TriageDecision.FALSE_POSITIVE);
    }

    @Test
    void ambiguousScore_inconclusive() {
        // CORPORATE 0.5 risk, structuring: score = 0.5*0.35 + 1.0*0.25 = 0.425
        var result = evaluator.evaluate(
                input("CORPORATE", 0.5, true, false, false, false, null));
        assertThat(result.decision()).isEqualTo(TriageDecision.INCONCLUSIVE);
    }

    @Test
    void cbrShiftsBorderline_toFalsePositive() {
        // Score slightly above FP threshold, CBR says FALSE_POSITIVE → push into FP range
        var cbr = new CbrPathAdvice(10, 0.8, 0.7, "FALSE_POSITIVE", 0.9, false);
        // CORPORATE 0.7 risk, no structuring: score = 0.7*0.35 = 0.245
        // Without CBR: just above 0.25 → INCONCLUSIVE
        // With CBR FP: FP threshold goes up → FALSE_POSITIVE
        var result = evaluator.evaluate(
                input("CORPORATE", 0.7, false, false, false, false, cbr));
        assertThat(result.decision()).isEqualTo(TriageDecision.FALSE_POSITIVE);
        assertThat(result.cbrThresholdAdjustment()).isNotNull();
    }

    @Test
    void cbrShiftsBorderline_toSarWarranted() {
        // Score slightly below SAR threshold, CBR says SAR → push into SAR range
        var cbr = new CbrPathAdvice(10, 0.8, 0.7, "SAR_WARRANTED", 0.9, false);
        // CORPORATE 0.9 risk, structuring: score = 0.9*0.35 + 1.0*0.25 = 0.565
        // Without CBR: below 0.6 → INCONCLUSIVE
        // With CBR SAR: SAR threshold drops → SAR_WARRANTED
        var result = evaluator.evaluate(
                input("CORPORATE", 0.9, true, false, false, false, cbr));
        assertThat(result.decision()).isEqualTo(TriageDecision.SAR_WARRANTED);
    }

    @Test
    void cbrCannotOverrideHardGate() {
        var cbr = new CbrPathAdvice(10, 0.9, 0.9, "FALSE_POSITIVE", 1.0, false);
        var result = evaluator.evaluate(
                input("PEP", 0.1, false, false, true, false, cbr));
        assertThat(result.decision()).isEqualTo(TriageDecision.SAR_WARRANTED);
        assertThat(result.hardGate()).isEqualTo(HardGate.SANCTIONS_HIT);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn -pl api -am test -Dtest=InvestigationTriageEvaluatorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure

- [ ] **Step 3: Implement InvestigationTriageEvaluator**

Create `api/src/main/java/io/casehub/aml/triage/InvestigationTriageEvaluator.java`:

```java
package io.casehub.aml.triage;

import io.casehub.aml.domain.TriageDecision;
import io.casehub.aml.domain.TriageInput;
import io.casehub.aml.domain.TriageResult;

public final class InvestigationTriageEvaluator {

    private final HardGateEvaluator hardGateEvaluator;
    private final RiskScorer riskScorer;
    private final CbrAdjuster cbrAdjuster;
    private final double sarThreshold;
    private final double fpThreshold;

    public InvestigationTriageEvaluator(double sarThreshold, double fpThreshold,
                                        double maxCbrAdjustment, double cbrMinConfidence) {
        this.hardGateEvaluator = new HardGateEvaluator();
        this.riskScorer = new RiskScorer();
        this.cbrAdjuster = new CbrAdjuster(maxCbrAdjustment, cbrMinConfidence);
        this.sarThreshold = sarThreshold;
        this.fpThreshold = fpThreshold;
    }

    public TriageResult evaluate(TriageInput input) {
        var gateResult = hardGateEvaluator.evaluate(input);
        if (gateResult.isPresent()) {
            return gateResult.get();
        }

        var scoring = riskScorer.score(input);
        var adjusted = cbrAdjuster.adjust(sarThreshold, fpThreshold, input.cbrPathAdvice());

        TriageDecision decision;
        String reason;
        if (scoring.score() >= adjusted.sarThreshold()) {
            decision = TriageDecision.SAR_WARRANTED;
            reason = String.format("Risk score %.3f >= SAR threshold %.3f", scoring.score(), adjusted.sarThreshold());
        } else if (scoring.score() <= adjusted.fpThreshold()) {
            decision = TriageDecision.FALSE_POSITIVE;
            reason = String.format("Risk score %.3f <= FP threshold %.3f", scoring.score(), adjusted.fpThreshold());
        } else {
            decision = TriageDecision.INCONCLUSIVE;
            reason = String.format("Risk score %.3f in ambiguous band (%.3f, %.3f)",
                    scoring.score(), adjusted.fpThreshold(), adjusted.sarThreshold());
        }

        return new TriageResult(decision, reason, scoring.score(),
                null, adjusted.adjustment(), scoring.factors());
    }
}
```

- [ ] **Step 4: Run tests to verify green**

Run: `mvn -pl api -am test -Dtest=InvestigationTriageEvaluatorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```
feat(#112): InvestigationTriageEvaluator — composed triage with gates, scoring, CBR adjustment

Refs #112
```

---

### Task 7: AmlActionType.INVESTIGATION_CLEARANCE + AmlTriagePolicyKeys

Action type for INCONCLUSIVE gate and preference keys for triage thresholds.

**Files:**
- Modify: `api/src/main/java/io/casehub/aml/domain/AmlActionType.java`
- Create: `app/src/main/java/io/casehub/aml/cbr/AmlTriagePolicyKeys.java`
- Test: `app/src/test/java/io/casehub/aml/engine/AmlActionRiskClassifierTest.java` (add test)

**Interfaces:**
- Produces: `AmlActionType.INVESTIGATION_CLEARANCE` — `GatePolicy.ALWAYS`, `reversible=true`, candidateGroups=[AML_COMPLIANCE]
- Produces: `AmlTriagePolicyKeys.SAR_THRESHOLD`, `.FALSE_POSITIVE_THRESHOLD`, `.MAX_CBR_ADJUSTMENT`, `.CBR_MIN_CONFIDENCE`

- [ ] **Step 1: Write failing test for INVESTIGATION_CLEARANCE**

Add to `AmlActionRiskClassifierTest.java`:

```java
@Test
void investigationClearance_alwaysGated_complianceGroup() {
    assertThat(AmlActionType.INVESTIGATION_CLEARANCE.gatePolicy())
            .isEqualTo(AmlActionType.GatePolicy.ALWAYS);
    assertThat(AmlActionType.INVESTIGATION_CLEARANCE.candidateGroups())
            .containsExactly(AmlGroups.AML_COMPLIANCE);
    assertThat(AmlActionType.INVESTIGATION_CLEARANCE.reversible()).isTrue();
    assertThat(AmlActionType.INVESTIGATION_CLEARANCE.actionType())
            .isEqualTo("investigation.clearance");
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl app -am test -Dtest=AmlActionRiskClassifierTest#investigationClearance_alwaysGated_complianceGroup -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure (enum constant doesn't exist)

- [ ] **Step 3: Add INVESTIGATION_CLEARANCE to AmlActionType**

Add after `LAW_ENFORCEMENT_REFERRAL`:

```java
INVESTIGATION_CLEARANCE(
    GatePolicy.ALWAYS, true,
    List.of(AmlGroups.AML_COMPLIANCE),
    "Investigation clearance on inconclusive evidence — compliance review required"),
```

- [ ] **Step 4: Create AmlTriagePolicyKeys**

Create `app/src/main/java/io/casehub/aml/cbr/AmlTriagePolicyKeys.java`:

```java
package io.casehub.aml.cbr;

import io.casehub.api.spi.routing.DoublePreference;
import io.casehub.platform.api.preferences.PreferenceKey;

public final class AmlTriagePolicyKeys {
    private static final String NS = "casehubio.aml.triage";

    public static final PreferenceKey<DoublePreference> SAR_THRESHOLD =
            new PreferenceKey<>(NS, "sar-threshold",
                    DoublePreference.of(0.6), DoublePreference::parse);

    public static final PreferenceKey<DoublePreference> FALSE_POSITIVE_THRESHOLD =
            new PreferenceKey<>(NS, "false-positive-threshold",
                    DoublePreference.of(0.25), DoublePreference::parse);

    public static final PreferenceKey<DoublePreference> MAX_CBR_ADJUSTMENT =
            new PreferenceKey<>(NS, "max-cbr-adjustment",
                    DoublePreference.of(0.15), DoublePreference::parse);

    public static final PreferenceKey<DoublePreference> CBR_MIN_CONFIDENCE =
            new PreferenceKey<>(NS, "cbr-min-confidence",
                    DoublePreference.of(0.3), DoublePreference::parse);

    private AmlTriagePolicyKeys() {}
}
```

- [ ] **Step 5: Run tests to verify green**

Run: `mvn -pl app -am test -Dtest=AmlActionRiskClassifierTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```
feat(#112): AmlActionType.INVESTIGATION_CLEARANCE + AmlTriagePolicyKeys

Refs #112
```

---

### Task 8: InvestigationTriageWorker rewrite + worker retrofit

Replace the stub, retrofit typed inputs in all workers that read specialist maps.

**Files:**
- Modify: `app/src/main/java/io/casehub/aml/cbr/InvestigationTriageWorker.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptor.java`
- Modify: `app/src/main/java/io/casehub/aml/engine/AmlInvestigationCaseHub.java`
- Modify: `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptorTest.java`

**Interfaces:**
- Consumes: `InvestigationTriageEvaluator`, `TriageInput`, `TriageResult`, `AmlTriagePolicyKeys`, `AmlActionType.INVESTIGATION_CLEARANCE`
- Produces: `InvestigationTriageWorker.create(ObjectMapper, PreferenceProvider) → Worker` (WorkerFunction.Sync)

- [ ] **Step 1: Read AmlInvestigationCaseDescriptorTest to understand existing structure**

Use IntelliJ: `ide_read_file` for `app/src/test/java/io/casehub/aml/engine/AmlInvestigationCaseDescriptorTest.java`

- [ ] **Step 2: Update AmlInvestigationCaseDescriptorTest for new constructor param**

The descriptor test will need an additional `PreferenceProvider` parameter. Use a mock or null
depending on what the test verifies (structural tests that don't execute workers can use null).

Also move `investigation-triage-agent` from FLOW_WORKERS to SYNC_WORKERS if the test classifies
workers by execution model.

- [ ] **Step 3: Rewrite InvestigationTriageWorker**

Replace `app/src/main/java/io/casehub/aml/cbr/InvestigationTriageWorker.java`:

```java
package io.casehub.aml.cbr;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.aml.domain.AmlActionType;
import io.casehub.aml.domain.CbrPathAdvice;
import io.casehub.aml.domain.EntityResolutionResult;
import io.casehub.aml.domain.OsintResult;
import io.casehub.aml.domain.PatternAnalysisResult;
import io.casehub.aml.domain.TriageDecision;
import io.casehub.aml.domain.TriageInput;
import io.casehub.aml.domain.TriageResult;
import io.casehub.aml.triage.InvestigationTriageEvaluator;
import io.casehub.api.spi.routing.DoublePreference;
import io.casehub.platform.api.preferences.PreferenceProvider;
import io.casehub.platform.api.preferences.Preferences;
import io.casehub.platform.api.preferences.SettingsScope;
import io.casehub.worker.api.PlannedAction;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.util.LinkedHashMap;
import java.util.Map;
import java.util.stream.Collectors;

public final class InvestigationTriageWorker {

    private InvestigationTriageWorker() {}

    public static Worker create(ObjectMapper objectMapper, PreferenceProvider preferenceProvider) {
        return Worker.builder()
                     .name("investigation-triage-agent")
                     .capabilityName("investigation-triage")
                     .function((Map<String, Object> input) -> {
                         var triageInput = deserializeInput(objectMapper, input);
                         var evaluator = buildEvaluator(preferenceProvider);
                         var result = evaluator.evaluate(triageInput);
                         return toWorkerResult(result);
                     })
                     .build();
    }

    private static TriageInput deserializeInput(ObjectMapper mapper, Map<String, Object> input) {
        var entity = mapper.convertValue(input.get("entityResolution"), EntityResolutionResult.class);
        var pattern = mapper.convertValue(input.get("patternAnalysis"), PatternAnalysisResult.class);
        var osint = mapper.convertValue(input.get("osintScreening"), OsintResult.class);
        CbrPathAdvice cbr = input.get("cbrPathAdvice") != null
                ? mapper.convertValue(input.get("cbrPathAdvice"), CbrPathAdvice.class) : null;
        return new TriageInput(entity, pattern, osint, cbr);
    }

    private static InvestigationTriageEvaluator buildEvaluator(PreferenceProvider provider) {
        try {
            Preferences prefs = provider.resolve(
                    SettingsScope.of("casehubio", "aml", "triage"));
            double sar = resolve(prefs, AmlTriagePolicyKeys.SAR_THRESHOLD, 0.6);
            double fp = resolve(prefs, AmlTriagePolicyKeys.FALSE_POSITIVE_THRESHOLD, 0.25);
            double maxAdj = resolve(prefs, AmlTriagePolicyKeys.MAX_CBR_ADJUSTMENT, 0.15);
            double minConf = resolve(prefs, AmlTriagePolicyKeys.CBR_MIN_CONFIDENCE, 0.3);
            return new InvestigationTriageEvaluator(sar, fp, maxAdj, minConf);
        } catch (Exception e) {
            return new InvestigationTriageEvaluator(0.6, 0.25, 0.15, 0.3);
        }
    }

    private static double resolve(Preferences prefs,
                                   io.casehub.platform.api.preferences.PreferenceKey<DoublePreference> key,
                                   double fallback) {
        var pref = prefs.getOrDefault(key);
        return pref != null ? pref.value() : fallback;
    }

    private static WorkerResult toWorkerResult(TriageResult result) {
        var map = new LinkedHashMap<String, Object>();
        map.put("decision", result.decision().name());
        map.put("reason", result.reason());
        map.put("riskScore", result.riskScore());
        if (result.hardGate() != null) map.put("hardGate", result.hardGate().name());
        if (result.cbrThresholdAdjustment() != null) map.put("cbrThresholdAdjustment", result.cbrThresholdAdjustment());
        if (!result.factors().isEmpty()) {
            map.put("factors", result.factors().stream()
                    .map(f -> Map.of("name", f.name(), "weight", f.weight(), "detail", f.detail()))
                    .toList());
        }

        if (result.decision() == TriageDecision.INCONCLUSIVE) {
            return WorkerResult.of(map, PlannedAction.of(
                    "Investigation clearance — inconclusive evidence requires compliance review",
                    AmlActionType.INVESTIGATION_CLEARANCE.actionType(),
                    Map.of(
                            "riskScore", String.valueOf(result.riskScore()),
                            "reason", result.reason(),
                            "factors", result.factors().stream()
                                    .map(f -> f.name() + "=" + f.weight())
                                    .collect(Collectors.joining(", ")))));
        }
        return WorkerResult.of(map);
    }
}
```

- [ ] **Step 4: Update AmlInvestigationCaseDescriptor**

Add `PreferenceProvider` as constructor parameter. Update `workers()` to pass it to
`InvestigationTriageWorker.create(objectMapper, preferenceProvider)`.

Retrofit typed inputs in sarDraftingWorkerJunior, sarDraftingWorkerSenior,
complianceReviewOpeningWorker, and entityResolutionWorker — replace map
extraction with `objectMapper.convertValue()` to domain records.

Update entityResolutionWorker to return SHELL_COMPANY/0.95 for
`HIGH_RISK_JURISDICTION` flag reason (needed for SAR hard-gate integration test):

```java
final boolean isPep = FlagReason.PEP_MATCH.name().equals(flagReason);
final boolean isHighRiskJurisdiction = FlagReason.HIGH_RISK_JURISDICTION.name().equals(flagReason);
final String entityType = isPep ? "PEP" : isHighRiskJurisdiction ? "SHELL_COMPANY" : "CORPORATE";
final double riskScore = isPep ? 0.87 : isHighRiskJurisdiction ? 0.95 : 0.35;
```

- [ ] **Step 5: Update AmlInvestigationCaseHub**

Inject `PreferenceProvider` and pass to descriptor constructor.

- [ ] **Step 6: Run the full test suite**

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 7: Commit**

```
feat(#112): investigation triage worker — rule-based triage replaces SAR_WARRANTED stub

Worker retrofit: typed domain record inputs via ObjectMapper.convertValue() in
entityResolution, sarDrafting, complianceReviewOpening, and triage workers.

Refs #112
```

---

### Task 9: Integration tests — all three triage paths

End-to-end verification that the engine correctly routes through all three
triage outcomes.

**Files:**
- Modify: `app/src/test/java/io/casehub/aml/cbr/InvestigationTriageFlowTest.java`

**Interfaces:**
- Consumes: `AmlEngineCoordinator.startInvestigation(SuspiciousTransaction)`, REST endpoint `/api/layer6/investigations/{id}`, `CaseInstanceCache`

- [ ] **Step 1: Retrofit existing SAR test**

Update `sarPath_triageStubReturnsSarWarranted_investigationCompletes` to verify structured
triage output. Assert `investigationTriage.hardGate` is present. Rename the test method to
remove "stub".

Use `FlagReason.HIGH_RISK_JURISDICTION` as the flag reason — entity resolution returns
entityType=SHELL_COMPANY (Task 8 stub update), triggering the SHELL_COMPANY hard gate →
SAR_WARRANTED with riskScore=1.0.

- [ ] **Step 2: Add FALSE_POSITIVE test path**

```java
@Test
@SuppressWarnings("unchecked")
void falsePositivePath_lowRisk_investigationCleared() {
    SuspiciousTransaction tx = new SuspiciousTransaction(
            "TXN-FP-" + UUID.randomUUID(),
            "ACC-FP-O-" + UUID.randomUUID(),
            "ACC-FP-D-" + UUID.randomUUID(),
            new BigDecimal("500"), "USD", Instant.now(), FlagReason.LARGE_VOLUME);

    UUID caseId = coordinator.startInvestigation(tx);
    drain(caseId);

    var instance = caseInstanceCache.get(caseId);
    var ctx = instance.getCaseContext();
    var triage = (Map<String, Object>) ctx.get("investigationTriage");
    assertThat(triage.get("decision")).isEqualTo("FALSE_POSITIVE");
    assertThat(ctx.get("sarNarrative")).isNull();
    assertThat(ctx.get("complianceTaskId")).isNull();
}
```

The key: `FlagReason.LARGE_VOLUME` with a small amount → entity resolution returns
entityType=CORPORATE, riskScore=0.35 → no structuring, no PEP, no sanctions → score
well below the FP threshold → FALSE_POSITIVE → `investigation-cleared` goal fires.

- [ ] **Step 3: Add INCONCLUSIVE test path with gate approval**

```java
@Test
@SuppressWarnings("unchecked")
void inconclusivePath_moderateRisk_gateApproved_investigationCleared() {
    // PEP_MATCH → entity resolution returns PEP/0.87 → score includes:
    // 0.87*0.35 (entity risk) + 0.20 (PEP type) + 0.05 (OSINT declined) = 0.5545
    // This is in the ambiguous band (0.25, 0.6) → INCONCLUSIVE
    SuspiciousTransaction tx = new SuspiciousTransaction(
            "TXN-INC-" + UUID.randomUUID(),
            "ACC-INC-O-" + UUID.randomUUID(),
            "ACC-INC-D-" + UUID.randomUUID(),
            new BigDecimal("15000"), "USD", Instant.now(), FlagReason.PEP_MATCH);

    UUID caseId = coordinator.startInvestigation(tx);
    // INCONCLUSIVE fires PlannedAction(INVESTIGATION_CLEARANCE) → gate WorkItem
    awaitAndApproveGate(caseId);
    drain(caseId);

    var instance = caseInstanceCache.get(caseId);
    var ctx = instance.getCaseContext();
    var triage = (Map<String, Object>) ctx.get("investigationTriage");
    assertThat(triage.get("decision")).isEqualTo("INCONCLUSIVE");
    assertThat(ctx.get("sarNarrative")).isNull();
    assertThat(ctx.get("complianceTaskId")).isNull();
}
```

Score breakdown: PEP_MATCH → entityType=PEP (riskScore=0.87), OSINT junior declines
(declined=true), pattern analysis returns structuringDetected=false. Score =
0.87×0.35 + 0×0.25 + 1.0×0.20 + 0.5×0.10 + 0×0.10 = 0.5545 → INCONCLUSIVE.
Note: the PEP_MATCH path also triggers `senior-analyst-required` binding — the test
must wait for that to complete before triage fires.

- [ ] **Step 4: Run integration tests**

Run: `mvn -pl app -am test -Dtest=InvestigationTriageFlowTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS for all three paths

- [ ] **Step 5: Run the full test suite**

Run: `mvn -pl app -am test -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — no regressions

- [ ] **Step 6: Commit**

```
test(#112): integration tests — SAR_WARRANTED, FALSE_POSITIVE, INCONCLUSIVE triage paths

Refs #112
```
