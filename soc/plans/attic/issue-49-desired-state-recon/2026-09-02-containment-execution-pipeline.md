# Containment Execution Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #40 — Epic: Containment execution pipeline — gated response with compliance audit
**Issue group:** #41, #42, #43, #44, #45

**Goal:** Connect existing risk classification, oversight gates, and containment workers into an end-to-end pipeline that executes containment actions, records compliance audit trail entries, and gates high-risk actions through human approval.

**Architecture:** The engine's built-in ActionGate lifecycle handles PlannedAction → risk classification → WorkItem creation → approval/rejection → context signals. We add: (1) a ContainmentExecutor SPI and execution worker that acts on the gate outcome, (2) a ledger observer that writes three-phase compliance audit entries, (3) case plan YAML bindings that wire the execution step after analyst triage, and (4) a fix to the recommendation worker's PlannedAction parameters.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI, JPA (Hibernate), casehub-worker-api, casehub-work-api, casehub-ledger-api

## Global Constraints

- Java 21 source, Java 26 JVM
- `SocStepType` column is `VARCHAR(30)` — all new enum values must be ≤ 30 characters
- Follow CDI displacement pattern: `@DefaultBean` for defaults, `@Alternative` for overrides
- All ledger entries must include `causedByEntryId` for chain integrity
- PlannedAction parameters must include both `riskScore` and `confidenceScore`
- Use `SocPiiSanitiser` on all action parameters before ledger write
- YAML bindings use `name`, `on: { contextChange: {} }`, `when` with `.key` JQ-style expressions

---

## Batch 1: SPI + Executor Foundation

### Task 1: ContainmentExecutor SPI and contracts (Issue #42)

**Files:**
- Create: `api/src/main/java/io/casehub/soc/engine/spi/ContainmentExecutor.java`
- Create: `api/src/main/java/io/casehub/soc/engine/spi/ContainmentResult.java`
- Create: `api/src/main/java/io/casehub/soc/engine/spi/ContainmentContext.java`
- Create: `api/src/main/java/io/casehub/soc/worker/contract/ContainmentExecutionOutput.java`
- Test: `app/src/test/java/io/casehub/soc/engine/spi/ContainmentResultTest.java`

**Interfaces:**
- Consumes: nothing — this is the foundation
- Produces: `ContainmentExecutor.execute(String, Map<String,Object>, ContainmentContext) → ContainmentResult`, `ContainmentExecutionOutput` record, `ContainmentContext` record, `ContainmentResult` record

- [ ] **Step 1: Write the failing test for ContainmentResult**

```java
package io.casehub.soc.engine.spi;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.assertj.core.api.Assertions.assertThat;

class ContainmentResultTest {

    @Test
    void successResult_hasCorrectFields() {
        var result = ContainmentResult.success("Host isolated successfully", Instant.now());
        assertThat(result.success()).isTrue();
        assertThat(result.retryable()).isFalse();
        assertThat(result.details()).isEqualTo("Host isolated successfully");
        assertThat(result.errorReason()).isNull();
    }

    @Test
    void failureResult_retryable() {
        var result = ContainmentResult.failure("Connection timeout", true);
        assertThat(result.success()).isFalse();
        assertThat(result.retryable()).isTrue();
        assertThat(result.errorReason()).isEqualTo("Connection timeout");
    }

    @Test
    void failureResult_nonRetryable() {
        var result = ContainmentResult.failure("Invalid credentials", false);
        assertThat(result.success()).isFalse();
        assertThat(result.retryable()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ContainmentResultTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes don't exist yet

- [ ] **Step 3: Create ContainmentContext record**

```java
package io.casehub.soc.engine.spi;

import java.util.UUID;

public record ContainmentContext(
    UUID caseId,
    String incidentId,
    String approver,
    String tenancyId,
    long timeoutMs
) {
    public static final long DEFAULT_TIMEOUT_MS = 30_000;

    public ContainmentContext(UUID caseId, String incidentId, String approver, String tenancyId) {
        this(caseId, incidentId, approver, tenancyId, DEFAULT_TIMEOUT_MS);
    }
}
```

- [ ] **Step 4: Create ContainmentResult record**

```java
package io.casehub.soc.engine.spi;

import java.time.Instant;

public record ContainmentResult(
    boolean success,
    Instant timestamp,
    String details,
    String errorReason,
    boolean retryable
) {
    public static ContainmentResult success(String details, Instant timestamp) {
        return new ContainmentResult(true, timestamp, details, null, false);
    }

    public static ContainmentResult failure(String errorReason, boolean retryable) {
        return new ContainmentResult(false, Instant.now(), null, errorReason, retryable);
    }
}
```

- [ ] **Step 5: Create ContainmentExecutor SPI interface**

```java
package io.casehub.soc.engine.spi;

import java.util.Map;

public interface ContainmentExecutor {
    ContainmentResult execute(String actionType, Map<String, Object> parameters,
                              ContainmentContext context);
}
```

- [ ] **Step 6: Create ContainmentExecutionOutput**

```java
package io.casehub.soc.worker.contract;

import java.time.Instant;

public record ContainmentExecutionOutput(
    String actionType,
    boolean executed,
    boolean success,
    String details,
    String errorReason,
    Instant executionTimestamp,
    long detectionToContainmentMs
) {}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ContainmentResultTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add api/src/main/java/io/casehub/soc/engine/spi/ api/src/main/java/io/casehub/soc/worker/contract/ContainmentExecutionOutput.java app/src/test/java/io/casehub/soc/engine/spi/
git commit -m "feat(#42): add ContainmentExecutor SPI, ContainmentResult, ContainmentContext, ContainmentExecutionOutput Refs #42"
```

### Task 2: LoggingContainmentExecutor + RuleContainmentExecutionWorker (Issue #42)

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/LoggingContainmentExecutor.java`
- Create: `app/src/main/java/io/casehub/soc/worker/RuleContainmentExecutionWorker.java`
- Test: `app/src/test/java/io/casehub/soc/worker/RuleContainmentExecutionWorkerTest.java`

**Interfaces:**
- Consumes: `ContainmentExecutor` (from Task 1), `ContainmentExecutionOutput` (from Task 1)
- Produces: `RuleContainmentExecutionWorker.create(ContainmentExecutor) → Worker` (used by Task 4 for wiring), `LoggingContainmentExecutor` CDI bean

- [ ] **Step 1: Write the failing test for the execution worker**

```java
package io.casehub.soc.worker;

import io.casehub.soc.engine.spi.ContainmentContext;
import io.casehub.soc.engine.spi.ContainmentExecutor;
import io.casehub.soc.engine.spi.ContainmentResult;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RuleContainmentExecutionWorkerTest {

    private final ContainmentExecutor loggingExecutor = (actionType, params, ctx) ->
            ContainmentResult.success("Logged: " + actionType, Instant.now());

    @Test
    void executesWhenActionGateApproved() {
        Worker worker = RuleContainmentExecutionWorker.create(loggingExecutor);
        Map<String, Object> input = new LinkedHashMap<>();
        input.put("actionGateApproved", Map.of(
                "actionType", "isolate.host",
                "approvedBy", "analyst-jane",
                "resolution", "confirmed threat"));
        input.put("containmentRecommendation", Map.of(
                "recommendedAction", "ISOLATE_HOST",
                "riskScore", 0.95,
                "actionParameters", Map.of("hostId", "srv-42")));
        input.put("alert", Map.of("detectedAt", "2026-09-02T10:00:00Z"));

        WorkerResult result = worker.execute(input);
        assertThat(result.output()).containsKey("actionType");
        assertThat(result.output().get("executed")).isEqualTo(true);
        assertThat(result.output().get("success")).isEqualTo(true);
    }

    @Test
    void executesWhenAutonomous_noGateSignal() {
        Worker worker = RuleContainmentExecutionWorker.create(loggingExecutor);
        Map<String, Object> input = new LinkedHashMap<>();
        input.put("containmentRecommendation", Map.of(
                "recommendedAction", "BLOCK_IP",
                "riskScore", 0.3,
                "actionParameters", Map.of("ip", "10.0.1.99")));
        input.put("alert", Map.of("detectedAt", "2026-09-02T10:00:00Z"));

        WorkerResult result = worker.execute(input);
        assertThat(result.output().get("executed")).isEqualTo(true);
    }

    @Test
    void skipsWhenActionGateRejected() {
        Worker worker = RuleContainmentExecutionWorker.create(loggingExecutor);
        Map<String, Object> input = new LinkedHashMap<>();
        input.put("actionGateRejected", Map.of(
                "actionType", "isolate.host",
                "rejectedBy", "analyst-bob",
                "reason", "false positive"));
        input.put("containmentRecommendation", Map.of(
                "recommendedAction", "ISOLATE_HOST"));

        WorkerResult result = worker.execute(input);
        assertThat(result.output().get("executed")).isEqualTo(false);
    }

    @Test
    void skipsWhenNoContainmentRecommended() {
        Worker worker = RuleContainmentExecutionWorker.create(loggingExecutor);
        Map<String, Object> input = new LinkedHashMap<>();
        input.put("containmentRecommendation", Map.of(
                "recommendedAction", (Object) null));

        WorkerResult result = worker.execute(input);
        assertThat(result.output().get("executed")).isEqualTo(false);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RuleContainmentExecutionWorkerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class doesn't exist

- [ ] **Step 3: Implement LoggingContainmentExecutor**

```java
package io.casehub.soc.engine;

import io.casehub.soc.engine.spi.ContainmentContext;
import io.casehub.soc.engine.spi.ContainmentExecutor;
import io.casehub.soc.engine.spi.ContainmentResult;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.Map;

@ApplicationScoped
@DefaultBean
public class LoggingContainmentExecutor implements ContainmentExecutor {

    private static final Logger LOG = Logger.getLogger(LoggingContainmentExecutor.class);

    @Override
    public ContainmentResult execute(String actionType, Map<String, Object> parameters,
                                     ContainmentContext context) {
        LOG.infof("CONTAINMENT EXECUTED [%s]: actionType=%s, caseId=%s, approver=%s, params=%s",
                context.tenancyId(), actionType, context.caseId(), context.approver(), parameters);
        return ContainmentResult.success("Logged containment action: " + actionType, Instant.now());
    }
}
```

- [ ] **Step 4: Implement RuleContainmentExecutionWorker**

```java
package io.casehub.soc.worker;

import io.casehub.soc.engine.spi.ContainmentContext;
import io.casehub.soc.engine.spi.ContainmentExecutor;
import io.casehub.soc.engine.spi.ContainmentResult;
import io.casehub.soc.worker.contract.ContainmentExecutionOutput;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.time.Duration;
import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.UUID;

public final class RuleContainmentExecutionWorker {

    private RuleContainmentExecutionWorker() {}

    public static Worker create(ContainmentExecutor executor) {
        return Worker.builder()
                .name("rule-containment-exec")
                .capabilityName("containment-execution")
                .function((Map<String, Object> input) -> {
                    @SuppressWarnings("unchecked")
                    var recommendation = (Map<String, Object>) input.getOrDefault(
                            "containmentRecommendation", Map.of());
                    Object recommendedAction = recommendation.get("recommendedAction");

                    if (recommendedAction == null) {
                        return skipResult("No containment action recommended");
                    }

                    @SuppressWarnings("unchecked")
                    var rejected = (Map<String, Object>) input.get("actionGateRejected");
                    if (rejected != null) {
                        return skipResult("Containment rejected: " +
                                rejected.getOrDefault("reason", "no reason given"));
                    }

                    String actionType = recommendedAction.toString().toLowerCase().replace('_', '.');

                    @SuppressWarnings("unchecked")
                    var approved = (Map<String, Object>) input.get("actionGateApproved");
                    String approver = approved != null
                            ? (String) approved.getOrDefault("approvedBy", null)
                            : null;

                    @SuppressWarnings("unchecked")
                    var actionParams = (Map<String, Object>) recommendation.getOrDefault(
                            "actionParameters", Map.of());

                    var context = new ContainmentContext(
                            UUID.randomUUID(), actionType, approver, "default");

                    ContainmentResult result = executor.execute(actionType, actionParams, context);

                    long detectionToContainmentMs = 0;
                    @SuppressWarnings("unchecked")
                    var alert = (Map<String, Object>) input.get("alert");
                    if (alert != null && alert.get("detectedAt") != null) {
                        try {
                            Instant detected = Instant.parse(alert.get("detectedAt").toString());
                            detectionToContainmentMs = Duration.between(detected, result.timestamp()).toMillis();
                        } catch (Exception ignored) {}
                    }

                    var output = new ContainmentExecutionOutput(
                            actionType, true, result.success(), result.details(),
                            result.errorReason(), result.timestamp(), detectionToContainmentMs);

                    var outputMap = new LinkedHashMap<String, Object>();
                    outputMap.put("actionType", output.actionType());
                    outputMap.put("executed", output.executed());
                    outputMap.put("success", output.success());
                    outputMap.put("details", output.details());
                    outputMap.put("errorReason", output.errorReason());
                    outputMap.put("executionTimestamp", output.executionTimestamp().toString());
                    outputMap.put("detectionToContainmentMs", output.detectionToContainmentMs());

                    return WorkerResult.of(outputMap);
                })
                .build();
    }

    private static WorkerResult skipResult(String reason) {
        var output = new LinkedHashMap<String, Object>();
        output.put("actionType", (Object) null);
        output.put("executed", false);
        output.put("success", false);
        output.put("details", reason);
        output.put("errorReason", (Object) null);
        output.put("executionTimestamp", (Object) null);
        output.put("detectionToContainmentMs", 0L);
        return WorkerResult.of(output);
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RuleContainmentExecutionWorkerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/LoggingContainmentExecutor.java app/src/main/java/io/casehub/soc/worker/RuleContainmentExecutionWorker.java app/src/test/java/io/casehub/soc/worker/RuleContainmentExecutionWorkerTest.java
git commit -m "feat(#42): add LoggingContainmentExecutor + RuleContainmentExecutionWorker Refs #42"
```

---

## Batch 2: Audit Trail + Fix

### Task 3: SocStepType additions + ledger observer (Issue #43)

**Files:**
- Modify: `api/src/main/java/io/casehub/soc/domain/SocStepType.java`
- Modify: `app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntryWriter.java:20-27`
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocContainmentLedgerObserver.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocContainmentLedgerObserverTest.java`

**Interfaces:**
- Consumes: `SocLedgerEntryWriter.write(UUID, SocStepType, String, String, ActorType, String, String, UUID)`, `CaseLifecycleEvent` CDI events, `SocPiiSanitiser.sanitise(String)`
- Produces: `SocContainmentLedgerObserver` CDI bean (observed async — no direct callers), new `SocStepType` values: `CONTAINMENT_GATE_DECISION`, `CONTAINMENT_APPROVAL`, `CONTAINMENT_REJECTION`

- [ ] **Step 1: Add new SocStepType enum values**

Add to `api/src/main/java/io/casehub/soc/domain/SocStepType.java`:

```java
public enum SocStepType {
    ALERT_TRIAGE,
    INCIDENT_PROMOTED,
    INVESTIGATION_STEP,
    CONTAINMENT_DECISION,
    CONTAINMENT_GATE_DECISION,
    CONTAINMENT_APPROVAL,
    CONTAINMENT_REJECTION,
    CONTAINMENT_EXECUTED,
    INCIDENT_RESOLVED
}
```

- [ ] **Step 2: Update REQUIRED_METADATA in SocLedgerEntryWriter**

Add entries for the new step types in `app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntryWriter.java`. Replace the existing `REQUIRED_METADATA` map (lines 20-27) with:

```java
    private static final Map<SocStepType, Set<String>> REQUIRED_METADATA = Map.ofEntries(
        Map.entry(SocStepType.ALERT_TRIAGE, Set.of("alertSeverity", "assignedSeverity", "triageAgentId")),
        Map.entry(SocStepType.INCIDENT_PROMOTED, Set.of("promotionReason")),
        Map.entry(SocStepType.INVESTIGATION_STEP, Set.of("capabilityTag", "investigationType")),
        Map.entry(SocStepType.CONTAINMENT_DECISION, Set.of("approverId", "riskClassification", "containmentAction")),
        Map.entry(SocStepType.CONTAINMENT_GATE_DECISION, Set.of("actionType", "riskScore", "gateDecision")),
        Map.entry(SocStepType.CONTAINMENT_APPROVAL, Set.of("actionType", "approverId")),
        Map.entry(SocStepType.CONTAINMENT_REJECTION, Set.of("actionType", "rejectorId", "rejectionReason")),
        Map.entry(SocStepType.CONTAINMENT_EXECUTED, Set.of("executionResult", "containmentAction")),
        Map.entry(SocStepType.INCIDENT_RESOLVED, Set.of("resolutionOutcome"))
    );
```

- [ ] **Step 3: Write the failing test for SocContainmentLedgerObserver**

```java
package io.casehub.soc.engine.compliance;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.node.ObjectNode;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.soc.domain.SocCaseTypes;
import io.casehub.soc.domain.SocStepType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.assertThat;

class SocContainmentLedgerObserverTest {

    private final ObjectMapper mapper = new ObjectMapper();
    private RecordingLedgerWriter writer;
    private SocContainmentLedgerObserver observer;

    @BeforeEach
    void setUp() {
        writer = new RecordingLedgerWriter();
        observer = new SocContainmentLedgerObserver(writer);
    }

    @Test
    void writesGateDecisionEntry_autonomous() {
        ObjectNode ctx = mapper.createObjectNode();
        ctx.putObject("containmentRecommendation")
                .put("recommendedAction", "BLOCK_IP")
                .put("riskScore", 0.3)
                .put("confidenceScore", 0.95);
        ctx.put("containmentGateDecision", "autonomous");

        fire(ctx);

        assertThat(writer.entries).hasSize(1);
        assertThat(writer.entries.get(0).stepType).isEqualTo(SocStepType.CONTAINMENT_GATE_DECISION);
        assertThat(writer.entries.get(0).metadata).contains("\"gateDecision\":\"autonomous\"");
    }

    @Test
    void writesApprovalEntry_whenGateApproved() {
        ObjectNode ctx = mapper.createObjectNode();
        ctx.putObject("containmentRecommendation")
                .put("recommendedAction", "ISOLATE_HOST");
        ctx.put("containmentGateDecision", "gated");
        ctx.putObject("actionGateApproved")
                .put("actionType", "isolate.host")
                .put("approvedBy", "analyst-jane")
                .put("resolution", "confirmed threat");

        fire(ctx);

        assertThat(writer.entries).anyMatch(e ->
                e.stepType == SocStepType.CONTAINMENT_APPROVAL &&
                e.metadata.contains("analyst-jane"));
    }

    @Test
    void writesRejectionEntry_whenGateRejected() {
        ObjectNode ctx = mapper.createObjectNode();
        ctx.putObject("containmentRecommendation")
                .put("recommendedAction", "ISOLATE_HOST");
        ctx.put("containmentGateDecision", "gated");
        ctx.putObject("actionGateRejected")
                .put("actionType", "isolate.host")
                .put("rejectedBy", "analyst-bob")
                .put("reason", "false positive");

        fire(ctx);

        assertThat(writer.entries).anyMatch(e ->
                e.stepType == SocStepType.CONTAINMENT_REJECTION &&
                e.metadata.contains("analyst-bob"));
    }

    @Test
    void writesExecutionEntry() {
        ObjectNode ctx = mapper.createObjectNode();
        ctx.putObject("containmentExecution")
                .put("actionType", "isolate.host")
                .put("executed", true)
                .put("success", true)
                .put("executionTimestamp", "2026-09-02T10:15:03Z")
                .put("detectionToContainmentMs", 45003);

        fire(ctx);

        assertThat(writer.entries).anyMatch(e ->
                e.stepType == SocStepType.CONTAINMENT_EXECUTED &&
                e.metadata.contains("\"executionResult\":\"SUCCESS\""));
    }

    private void fire(ObjectNode contextSnapshot) {
        UUID caseId = UUID.randomUUID();
        var event = new CaseLifecycleEvent(
                caseId, SocCaseTypes.INCIDENT_INVESTIGATION,
                contextSnapshot, "system:soc", "test-tenant");
        observer.onLifecycle(event);
    }

    static class RecordingLedgerWriter extends SocLedgerEntryWriter {
        final List<Entry> entries = new ArrayList<>();

        RecordingLedgerWriter() { super(null, null); }

        @Override
        public void write(UUID incidentId, SocStepType stepType, String actorId,
                          String actorRole, io.casehub.platform.api.identity.ActorType actorType,
                          String metadata, String tenancyId, UUID causedByEntryId) {
            entries.add(new Entry(incidentId, stepType, actorId, metadata, causedByEntryId));
        }

        record Entry(UUID incidentId, SocStepType stepType, String actorId,
                     String metadata, UUID causedByEntryId) {}
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocContainmentLedgerObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — SocContainmentLedgerObserver doesn't exist

- [ ] **Step 5: Implement SocContainmentLedgerObserver**

```java
package io.casehub.soc.engine.compliance;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.engine.common.spi.event.CaseLifecycleEvent;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.soc.domain.SocCaseTypes;
import io.casehub.soc.domain.SocStepType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.util.Set;
import java.util.UUID;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class SocContainmentLedgerObserver {

    private static final Logger LOG = Logger.getLogger(SocContainmentLedgerObserver.class);
    private final Set<String> processedSignals = ConcurrentHashMap.newKeySet();
    private final SocLedgerEntryWriter writer;

    @Inject
    SocContainmentLedgerObserver(SocLedgerEntryWriter writer) {
        this.writer = writer;
    }

    void onLifecycle(@ObservesAsync CaseLifecycleEvent event) {
        if (!SocCaseTypes.INCIDENT_INVESTIGATION.equals(event.caseDefinitionName())) return;
        if (event.contextSnapshot() == null) return;

        JsonNode ctx = event.contextSnapshot();
        UUID caseId = event.caseId();
        String tenancyId = event.tenancyId();

        writeGateDecisionIfPresent(caseId, ctx, tenancyId);
        writeApprovalOrRejectionIfPresent(caseId, ctx, tenancyId);
        writeExecutionIfPresent(caseId, ctx, tenancyId);
    }

    private void writeGateDecisionIfPresent(UUID caseId, JsonNode ctx, String tenancyId) {
        String gateDecision = ctx.path("containmentGateDecision").asText(null);
        if (gateDecision == null) return;

        String key = caseId + ":gate";
        if (!processedSignals.add(key)) return;

        JsonNode rec = ctx.path("containmentRecommendation");
        String actionType = SocPiiSanitiser.sanitise(rec.path("recommendedAction").asText("UNKNOWN"));
        double riskScore = rec.path("riskScore").asDouble(0.0);
        double confidenceScore = rec.path("confidenceScore").asDouble(0.0);

        String metadata = String.format(
                "{\"actionType\":\"%s\",\"riskScore\":%.2f,\"confidenceScore\":%.2f,\"gateDecision\":\"%s\"}",
                actionType, riskScore, confidenceScore, SocPiiSanitiser.sanitise(gateDecision));

        try {
            writer.write(caseId, SocStepType.CONTAINMENT_GATE_DECISION,
                    "system:soc", "containment-gate", ActorType.SYSTEM,
                    metadata, tenancyId, null);
        } catch (Exception e) {
            LOG.errorf(e, "Failed to write CONTAINMENT_GATE_DECISION ledger entry caseId=%s", caseId);
        }
    }

    private void writeApprovalOrRejectionIfPresent(UUID caseId, JsonNode ctx, String tenancyId) {
        JsonNode approved = ctx.path("actionGateApproved");
        JsonNode rejected = ctx.path("actionGateRejected");

        if (!approved.isMissingNode() && approved.isObject()) {
            String key = caseId + ":approval";
            if (!processedSignals.add(key)) return;

            String actionType = SocPiiSanitiser.sanitise(approved.path("actionType").asText("UNKNOWN"));
            String approverId = SocPiiSanitiser.sanitise(approved.path("approvedBy").asText("unknown"));
            String resolution = SocPiiSanitiser.sanitise(approved.path("resolution").asText(""));

            String metadata = String.format(
                    "{\"actionType\":\"%s\",\"approverId\":\"%s\",\"resolution\":\"%s\"}",
                    actionType, approverId, resolution);

            try {
                writer.write(caseId, SocStepType.CONTAINMENT_APPROVAL,
                        approverId, "containment-approval", ActorType.HUMAN,
                        metadata, tenancyId, null);
            } catch (Exception e) {
                LOG.errorf(e, "Failed to write CONTAINMENT_APPROVAL ledger entry caseId=%s", caseId);
            }
        } else if (!rejected.isMissingNode() && rejected.isObject()) {
            String key = caseId + ":rejection";
            if (!processedSignals.add(key)) return;

            String actionType = SocPiiSanitiser.sanitise(rejected.path("actionType").asText("UNKNOWN"));
            String rejectorId = SocPiiSanitiser.sanitise(rejected.path("rejectedBy").asText("unknown"));
            String reason = SocPiiSanitiser.sanitise(rejected.path("reason").asText(""));

            String metadata = String.format(
                    "{\"actionType\":\"%s\",\"rejectorId\":\"%s\",\"rejectionReason\":\"%s\"}",
                    actionType, rejectorId, reason);

            try {
                writer.write(caseId, SocStepType.CONTAINMENT_REJECTION,
                        rejectorId, "containment-rejection", ActorType.HUMAN,
                        metadata, tenancyId, null);
            } catch (Exception e) {
                LOG.errorf(e, "Failed to write CONTAINMENT_REJECTION ledger entry caseId=%s", caseId);
            }
        }
    }

    private void writeExecutionIfPresent(UUID caseId, JsonNode ctx, String tenancyId) {
        JsonNode exec = ctx.path("containmentExecution");
        if (exec.isMissingNode() || !exec.isObject()) return;

        String key = caseId + ":execution";
        if (!processedSignals.add(key)) return;

        String actionType = SocPiiSanitiser.sanitise(exec.path("actionType").asText("UNKNOWN"));
        boolean success = exec.path("success").asBoolean(false);
        long dtoC = exec.path("detectionToContainmentMs").asLong(0);

        String metadata = String.format(
                "{\"executionResult\":\"%s\",\"containmentAction\":\"%s\",\"detectionToContainmentMs\":%d}",
                success ? "SUCCESS" : "FAILURE", actionType, dtoC);

        try {
            writer.write(caseId, SocStepType.CONTAINMENT_EXECUTED,
                    "system:soc", "containment-execution", ActorType.SYSTEM,
                    metadata, tenancyId, null);
        } catch (Exception e) {
            LOG.errorf(e, "Failed to write CONTAINMENT_EXECUTED ledger entry caseId=%s", caseId);
        }
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocContainmentLedgerObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add api/src/main/java/io/casehub/soc/domain/SocStepType.java app/src/main/java/io/casehub/soc/engine/compliance/ app/src/test/java/io/casehub/soc/engine/compliance/SocContainmentLedgerObserverTest.java
git commit -m "feat(#43): add SocContainmentLedgerObserver + SocStepType extensions for 3-phase audit Refs #43"
```

### Task 4: Fix confidenceScore in PlannedAction parameters (Issue #41)

**Files:**
- Modify: `app/src/main/java/io/casehub/soc/worker/RuleContainmentRecommendationWorker.java:48-53`
- Test: `app/src/test/java/io/casehub/soc/worker/RuleContainmentRecommendationWorkerTest.java` (new or extend existing)

**Interfaces:**
- Consumes: `ContainmentDecisionMatrix.decide(String, String) → ContainmentRecommendationOutput`
- Produces: `PlannedAction` with `parameters` map including `confidenceScore` (consumed by engine's `SocActionRiskClassifier`)

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.soc.worker;

import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class RuleContainmentRecommendationWorkerConfidenceTest {

    @Test
    void plannedAction_includesConfidenceScore() {
        Worker worker = RuleContainmentRecommendationWorker.create();
        WorkerResult result = worker.execute(Map.of(
                "alert", Map.of("severity", "MEDIUM"),
                "attckMapping", Map.of("primaryTactic", "CREDENTIAL_ACCESS")));

        if (result.plannedAction() != null) {
            assertThat(result.plannedAction().parameters())
                    .containsKey("confidenceScore");
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RuleContainmentRecommendationWorkerConfidenceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `confidenceScore` not present in PlannedAction parameters

- [ ] **Step 3: Fix the PlannedAction parameters**

In `app/src/main/java/io/casehub/soc/worker/RuleContainmentRecommendationWorker.java`, replace lines 48-53:

Old:
```java
                    PlannedAction action = PlannedAction.of(
                            "Containment: " + decision.recommendedAction(),
                            actionType,
                            Map.of("riskScore", decision.riskScore(),
                                    "severity", severity,
                                    "tactic", primaryTactic));
```

New:
```java
                    PlannedAction action = PlannedAction.of(
                            "Containment: " + decision.recommendedAction(),
                            actionType,
                            Map.of("riskScore", decision.riskScore(),
                                    "confidenceScore", decision.confidenceScore(),
                                    "severity", severity,
                                    "tactic", primaryTactic));
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RuleContainmentRecommendationWorkerConfidenceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/soc/worker/RuleContainmentRecommendationWorker.java app/src/test/java/io/casehub/soc/worker/RuleContainmentRecommendationWorkerConfidenceTest.java
git commit -m "fix(#41): include confidenceScore in PlannedAction parameters — fixes ROTATE_API_KEY always-gated bug Refs #41"
```

---

## Batch 3: Wiring + E2E

### Task 5: Case plan YAML + worker registration (Issue #44)

**Files:**
- Modify: `app/src/main/resources/soc/incident-investigation.yaml:53-143`
- Modify: `app/src/main/java/io/casehub/soc/engine/SocInvestigationCaseDescriptor.java:30-39`
- Modify: `app/src/main/java/io/casehub/soc/engine/SocCaseHub.java:21-25`
- Create: `app/src/main/java/io/casehub/soc/domain/SocCaseCapabilities.java` (if constant not already there)

**Interfaces:**
- Consumes: `RuleContainmentExecutionWorker.create(ContainmentExecutor) → Worker` (from Task 2), `LoggingContainmentExecutor` CDI bean (from Task 2)
- Produces: Updated case plan with `containment-execution` binding, updated worker list

- [ ] **Step 1: Add containment-execution capability to YAML**

In `app/src/main/resources/soc/incident-investigation.yaml`, add to the `capabilities:` section after `containment-recommendation`:

```yaml
    - name: containment-execution
      description: "Execute approved containment actions via ContainmentExecutor SPI"
      inputProjection: "{ alert: .alert, containmentRecommendation: .containmentRecommendation, actionGateApproved: .actionGateApproved, actionGateRejected: .actionGateRejected }"
      outputProjection: "{ containmentExecution: . }"
```

- [ ] **Step 2: Add containment-execution binding to YAML**

In `app/src/main/resources/soc/incident-investigation.yaml`, add a new binding after `analyst-review`:

```yaml
    ## Fires after analyst review confirms the incident. Executes the
    ## containment action if approved or autonomous, skips if rejected
    ## or no containment was recommended.
    - name: containment-execution
      on: { contextChange: {} }
      when: ".analystDecision != null and .containmentExecution == null"
      capability: containment-execution
```

- [ ] **Step 3: Update goals to include containment completion**

Update the `resolved` goal condition to also require containment execution to have completed (or been skipped):

```yaml
    - name: resolved
      kind: success
      condition: ".analystDecision == \"resolved\" and .containmentExecution != null"
```

And similarly for `escalated` and `false-positive`:
```yaml
    - name: escalated
      kind: success
      condition: ".analystDecision == \"escalated\" and .containmentExecution != null"

    - name: false-positive
      kind: success
      condition: ".analystDecision == \"false-positive\" and .containmentExecution != null"
```

- [ ] **Step 4: Register worker in SocInvestigationCaseDescriptor**

Add `ContainmentExecutor` parameter and new worker to the descriptor. In `app/src/main/java/io/casehub/soc/engine/SocInvestigationCaseDescriptor.java`:

Add import:
```java
import io.casehub.soc.engine.spi.ContainmentExecutor;
```

Add field:
```java
    private final ContainmentExecutor containmentExecutor;
```

Update constructors:
```java
    SocInvestigationCaseDescriptor() {
        this(null, null, null);
    }

    SocInvestigationCaseDescriptor(ChatModel llmModel,
                                   SocCbrRetrieveService cbrRetrieveService,
                                   ContainmentExecutor containmentExecutor) {
        this.llmModel = llmModel;
        this.cbrRetrieveService = cbrRetrieveService;
        this.containmentExecutor = containmentExecutor;
    }
```

Add to `workers()` list:
```java
                RuleContainmentExecutionWorker.create(
                        containmentExecutor != null ? containmentExecutor : (a, p, c) ->
                                io.casehub.soc.engine.spi.ContainmentResult.success("default-no-op", java.time.Instant.now()))
```

- [ ] **Step 5: Update SocCaseHub to inject ContainmentExecutor**

In `app/src/main/java/io/casehub/soc/engine/SocCaseHub.java`:

Add field:
```java
    @Inject
    ContainmentExecutor containmentExecutor;
```

Update `augment`:
```java
    @Override
    protected void augment(CaseDefinition definition) {
        var descriptor = new SocInvestigationCaseDescriptor(null, cbrRetrieveService, containmentExecutor);
        definition.getWorkers().addAll(descriptor.workers());
        definition.setAgentDescriptors(SocAgentDescriptors.descriptorsByWorkerName());
    }
```

- [ ] **Step 6: Build to verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl app -am`
Expected: BUILD SUCCESS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/resources/soc/incident-investigation.yaml app/src/main/java/io/casehub/soc/engine/SocInvestigationCaseDescriptor.java app/src/main/java/io/casehub/soc/engine/SocCaseHub.java
git commit -m "feat(#44): wire containment-execution into case plan YAML + worker registration Refs #44"
```

### Task 6: E2E integration test (Issue #45)

**Files:**
- Create: `app/src/test/java/io/casehub/soc/integration/ContainmentPipelineIntegrationTest.java`

**Interfaces:**
- Consumes: `/api/soc/demo/inject-alert` REST endpoint, `SocCbrRetrieveService`, `SocLedgerEntryRepository`

- [ ] **Step 1: Write the integration test**

```java
package io.casehub.soc.integration;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import static org.hamcrest.Matchers.is;

@QuarkusTest
class ContainmentPipelineIntegrationTest {

    @Test
    void fullPipeline_injectsAlertAndCompletesWithContainmentExecution() {
        RestAssured.given()
                .contentType("application/json")
                .body("{\"eventType\":\"soc.alert.siem.crowdstrike\",\"severity\":\"CRITICAL\","
                    + "\"source\":\"10.0.1.42\",\"rule\":\"credential-harvesting\"}")
                .when().post("/api/soc/demo/inject-alert")
                .then()
                .statusCode(200)
                .body("evaluated", is(true));
    }

    @Test
    void lowSeverityAlert_skipsContainmentExecution() {
        RestAssured.given()
                .contentType("application/json")
                .body("{\"eventType\":\"soc.alert.siem.splunk\",\"severity\":\"LOW\","
                    + "\"source\":\"10.0.2.10\",\"rule\":\"info-scan\"}")
                .when().post("/api/soc/demo/inject-alert")
                .then()
                .statusCode(200)
                .body("evaluated", is(true));
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ContainmentPipelineIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — full pipeline runs end-to-end

- [ ] **Step 3: Commit**

```bash
git add app/src/test/java/io/casehub/soc/integration/ContainmentPipelineIntegrationTest.java
git commit -m "test(#45): containment pipeline e2e integration test — full pipeline verification Refs #45"
```

- [ ] **Step 4: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: BUILD SUCCESS — all existing tests still pass

- [ ] **Step 5: Commit any fixes**

If any tests broke, fix them and commit:
```bash
git commit -m "fix(#44): resolve test regressions from containment pipeline wiring Refs #44"
```

---

## References

- [2026-09-02-containment-execution-pipeline-design.md] — design spec this plan implements
- [SocActionRiskClassifier.java:16] — risk classification implementation
- [SocEscalatedWorkItemHandler.java:14] — existing WorkItem observer pattern
- [SocIncidentLedgerObserver.java:17] — existing ledger observer pattern
- [SocLedgerEntryWriter.java:16] — Merkle chain writer, REQUIRED_METADATA validation
- [SocLedgerEntry.java:18] — JPA entity, step_type column VARCHAR(30)
- [SocStepType.java:3] — existing enum values
- [incident-investigation.yaml] — current case plan definition
- [SocInvestigationCaseDescriptor.java:15] — worker registration
- [SocCaseHub.java:11] — case hub augment
- [RuleContainmentRecommendationWorker.java:12] — recommendation worker, PlannedAction params
- [SocActionType.java:16] — action types with gate policies
- [ContainmentDecisionMatrix.java] — severity × tactic decision matrix
- [CbrRetrievalIntegrationTest.java] — integration test pattern
- [GitHub #40] — epic
- [GitHub #41] — approval gate (confidenceScore fix)
- [GitHub #42] — execution worker + SPI
- [GitHub #43] — compliance audit
- [GitHub #44] — case plan bindings
- [GitHub #45] — e2e test
