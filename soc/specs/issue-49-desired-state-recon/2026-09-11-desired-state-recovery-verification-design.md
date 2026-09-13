# Desired-State Recovery Verification — Design Spec

**Epic:** #49 — Desired-state reconciliation — post-containment recovery verification
**Date:** 2026-09-11
**Revision:** 1 — incorporates Standard decision review findings (R1-01 through R2-02)

---

## Problem

The containment execution pipeline (epic #40) logs actions and records audit entries, but never verifies the action took effect. An "isolate host" command that silently fails leaves the SOC believing the threat is contained when it isn't. DORA requires demonstrable recovery verification — not just "we sent the command."

## Approach

Wire casehub-desiredstate into the SOC containment recovery phase. After containment execution completes, a new case plan binding constructs a per-incident DesiredStateGraph representing the expected post-containment state, starts a reconciliation loop that periodically verifies actual system state against the expectation, and signals the case plan when verification completes or times out.

### Key architectural decisions

- **Per-case reconciliation scope (D1):** One graph per incident case, `tenancyId` = `case:<caseId>`. Lifecycle tied to case closure.
- **Detection, not remediation (D8):** SOC NodeProvisioners return `Failed` — the loop detects divergence, humans decide whether to re-execute.
- **Async verification (D7):** Worker starts reconciliation and returns immediately. GlobalReconciliationListener signals the case plan via `CaseHubRuntime.signal()` on convergence/divergence/timeout.
- **Direct graph construction (D10):** No GoalCompiler — the mapping from execution output to graph is direct.

## Pipeline Flow

Extends the incident-investigation case plan:

```
... → containment-execution → recovery-verification-start → (async reconciliation) → recovery-verification-result → ...
```

### Case Plan YAML Changes

```yaml
# Recovery verification — starts reconciliation after containment executes
- name: recovery-verification-start
  on: { contextChange: {} }
  when: ".containmentExecution != null and .containmentExecution.executed == true and .recoveryVerification == null"
  capability: recovery-verification-start

# Recovery verification result — fires when async reconciliation signals outcome
- name: recovery-verification-result
  on: { contextChange: {} }
  when: ".recoveryVerification != null and .recoveryVerification.status != \"started\""
  capability: recovery-verification-result
```

### Three verification paths

**Converged (all actions verified):**
```
execution completes → worker builds graph, starts LifecycleManager → reconciliation loop polls actual state
→ all nodes PRESENT → listener writes per-node RECOVERY_VERIFIED ledger entries
→ listener signals case context { status: "verified" } → result binding fires
```

**Diverged (timeout with unverified actions):**
```
execution completes → worker builds graph → reconciliation detects ABSENT/DRIFTED
→ ThresholdFaultPolicy: tier 1 retries, tier 2 adds review node → HumanNodeHandler creates WorkItem
→ timeout expires → listener signals { status: "diverged" } → result binding fires
→ reconciliation loop continues (SOC manager WorkItem still active)
```

**Partially verified (some actions verified before timeout):**
```
execution completes → worker builds graph → some nodes converge (per-node ledger entries written)
→ remaining nodes diverge → timeout expires → listener signals { status: "partially_verified" }
→ result binding fires
```

**No containment (execution skipped):**
No recovery-verification binding fires — the `executed == true` guard prevents activation.

### Single-action vs multi-action

The current containment pipeline executes one action per incident (`RuleContainmentExecutionWorker` processes a single `recommendedAction`). The recovery verification graph will therefore contain exactly one node. The `partially_verified` status is unreachable today — it exists to support future multi-action containment (e.g., `ISOLATE_HOST` + `BLOCK_IP` in a single incident). The worker's graph construction handles both cases identically: iterate executed actions, create one node per action.

## New Components

### api/ module

#### SocContainmentNodeTypes

Constants for containment verification node types.

```java
package io.casehub.soc.domain;

import io.casehub.desiredstate.api.NodeType;

public final class SocContainmentNodeTypes {
    public static final NodeType ISOLATE_HOST = NodeType.of("soc:isolate-host");
    public static final NodeType BLOCK_IP = NodeType.of("soc:block-ip");
    public static final NodeType BLOCK_DOMAIN = NodeType.of("soc:block-domain");
    public static final NodeType REVOKE_CREDENTIALS = NodeType.of("soc:revoke-credentials");
    public static final NodeType ROTATE_API_KEY = NodeType.of("soc:rotate-api-key");
    public static final NodeType DISABLE_USER_ACCOUNT = NodeType.of("soc:disable-user-account");
    public static final NodeType NETWORK_SEGMENTATION = NodeType.of("soc:network-segmentation");
    public static final NodeType WIPE_ENDPOINT = NodeType.of("soc:wipe-endpoint");

    private SocContainmentNodeTypes() {}
}
```

**Module:** `api/`
**Package:** `io.casehub.soc.domain`

#### SocContainmentNodeSpec (sealed interface)

Per-action NodeSpec implementations carrying verification parameters from the containment execution output.

```java
package io.casehub.soc.domain;

import io.casehub.desiredstate.api.NodeSpec;

public sealed interface SocContainmentNodeSpec extends NodeSpec {

    record IsolateHostSpec(String hostname, String edpAgentId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.ISOLATE_HOST; }
    }
    record BlockIpSpec(String ipAddress, String firewallRuleId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.BLOCK_IP; }
    }
    record BlockDomainSpec(String domain, String proxyRuleId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.BLOCK_DOMAIN; }
    }
    record RevokeCredentialsSpec(String credentialId, String iamProvider) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.REVOKE_CREDENTIALS; }
    }
    record RotateApiKeySpec(String keyId, String serviceId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.ROTATE_API_KEY; }
    }
    record DisableUserAccountSpec(String userId, String iamProvider) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.DISABLE_USER_ACCOUNT; }
    }
    record NetworkSegmentationSpec(String segmentId, String firewallRuleId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.NETWORK_SEGMENTATION; }
    }
    record WipeEndpointSpec(String hostname, String edpAgentId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.WIPE_ENDPOINT; }
    }
}
```

`ENABLE_ENHANCED_LOGGING` is excluded — it is a monitoring action (`GatePolicy.NEVER`, `reversible=true`), not a containment action. No post-action verification needed.
```

**Module:** `api/`
**Package:** `io.casehub.soc.domain`

#### RecoveryVerificationOutput

Worker output contract for recovery-verification-start.

```java
package io.casehub.soc.worker.contract;

import java.time.Instant;

public record RecoveryVerificationOutput(
    String status,
    Instant startedAt,
    String tenancyId
) {}
```

**Module:** `api/`
**Package:** `io.casehub.soc.worker.contract`

#### RecoveryVerificationResultOutput

Worker output contract for recovery-verification-result.

```java
package io.casehub.soc.worker.contract;

import java.time.Instant;
import java.util.List;

public record RecoveryVerificationResultOutput(
    String status,
    List<String> verifiedNodes,
    List<String> divergedNodes,
    Instant completedAt,
    long executionToVerificationMs
) {}
```

**Module:** `api/`
**Package:** `io.casehub.soc.worker.contract`

#### SocDivergenceReviewSpec

NodeSpec for review nodes created by ThresholdFaultPolicy escalation.

```java
package io.casehub.soc.domain;

import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;

public record SocDivergenceReviewSpec(
    NodeId faultedNodeId,
    String faultType,
    String faultMessage
) implements NodeSpec {

    public static final NodeType REVIEW_TYPE = NodeType.of("soc:divergence-review");

    @Override public NodeType nodeType() { return REVIEW_TYPE; }

    @Override
    public HumanGating humanGating() {
        return HumanGating.ALL;
    }

    public static SocDivergenceReviewSpec fromFaultEvent(FaultEvent event, DesiredStateGraph current) {
        return new SocDivergenceReviewSpec(event.node(), event.type().name(), event.detail());
    }
}
```

**Module:** `api/`
**Package:** `io.casehub.soc.domain`

### app/ module

#### RuleRecoveryVerificationWorker

Worker for `recovery-verification-start` capability. Reads containment execution output from case context, builds a DesiredStateGraph, starts a LifecycleManager, and returns.

```java
package io.casehub.soc.worker;

// Worker.builder() function
// Reads case context: containmentExecution (actionType, executed, success, details)
// Maps actionType → SocContainmentNodeType + SocContainmentNodeSpec
// Builds DesiredStateGraph with one node per executed action
// Creates LifecycleManager with single phase, CompletionCondition.allPresent()
// tenancyId = "case:" + caseId
// Calls lifecycleManager.start()
// Returns WorkerResult with RecoveryVerificationOutput { status: "started", startedAt, tenancyId }
```

**Module:** `app/`
**Package:** `io.casehub.soc.worker`

#### SocContainmentActualStateAdapter

Default `@DefaultBean` adapter — mirrors LoggingContainmentExecutor. Reports PRESENT for all known node types. Real adapters (EDR, firewall, IAM) displace via `@Alternative`.

```java
package io.casehub.soc.engine;

// @DefaultBean @ApplicationScoped
// implements ActualStateAdapter
// handledTypes() returns all SocContainmentNodeTypes
// readActual() returns PRESENT for every node in the desired graph
// Production startup health check warns when this default is active in non-dev profiles
```

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

#### SocContainmentNodeProvisioner

Returns `Failed` for all containment types — SOC reconciliation is detection-only.

```java
package io.casehub.soc.engine;

// @ApplicationScoped
// implements NodeProvisioner
// handledTypes() returns all SocContainmentNodeTypes
// provision() returns ProvisionResult.Failed("re-provisioning disabled — containment divergence requires human review")
// deprovision() returns DeprovisionResult.Failed("containment node removal requires case closure")
```

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

#### SocContainmentFaultPolicy

ThresholdFaultPolicy configured for containment verification escalation.

```java
package io.casehub.soc.engine;

// @Produces @ApplicationScoped
// ThresholdFaultPolicy.builder()
//   .faultTypes(Set.of(PROVISION_FAILED, NODE_DEGRADED))
//   .nodeTypes(all SocContainmentNodeTypes)
//   .tier(1, TypedFaultPolicy.of(NodeType.of("soc:retry-noop"), (t, f, g, a) -> List.of()))
//   .tier(2, addReviewNode(SocDivergenceReviewSpec::fromFaultEvent))
//   .namespace("soc-containment-verification")
//   .build()
// ReviewSpecFactory creates SocDivergenceReviewSpec with faulted node context
```

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

#### SocRecoveryVerificationListener

GlobalReconciliationListener — detects convergence/divergence/timeout, writes ledger entries, signals case context.

```java
package io.casehub.soc.engine.compliance;

// @ApplicationScoped
// implements GlobalReconciliationListener
// Injected: CaseHubRuntime, SocLedgerEntryWriter
// Maintains internal state: Map<String, VerificationState> tracking per-tenant startedAt, already-verified nodes
// onReconciliationCycleCompleted(tenancyId, desired, actual):
//   1. Filter: only process tenancyId starting with "case:"
//   2. Retrieve startedAt: CaseHubRuntime.query(caseId, "recoveryVerification.startedAt")
//      (cached in VerificationState after first retrieval to avoid per-cycle query)
//   3. Per-node convergence: for each node transitioning to PRESENT (not already recorded):
//      - Write RECOVERY_VERIFIED ledger entry via SocLedgerEntryWriter
//      - causedByEntryId links to CONTAINMENT_EXECUTED entry
//      - Record execution→convergence time (DORA timeline)
//   4. All-converged check: if all nodes PRESENT:
//      - CaseHubRuntime.signal(caseId, "recoveryVerification",
//          { status: "verified", convergenceTime: ..., verifiedNodes: [...] })
//   5. Timeout check: if elapsed > timeout (severity-dependent, from SocPreferences):
//      - If some converged: signal { status: "partially_verified", verifiedNodes, divergedNodes }
//      - If none converged: signal { status: "diverged", divergedNodes }
//   6. Neither → continue (next cycle will re-check)
// onTenantStopped(tenancyId): remove VerificationState entry
//
// Restart recovery: on first cycle after restart (VerificationState missing for a "case:" tenancy),
// re-derive already-verified nodes by querying SocLedgerEntryRepository for existing RECOVERY_VERIFIED
// entries matching this caseId. This prevents duplicate ledger entries after service restart.
```

**Module:** `app/`
**Package:** `io.casehub.soc.engine.compliance`

#### SocRecoveryVerificationResultWorker

Worker for `recovery-verification-result` capability. Records the verification outcome.

```java
package io.casehub.soc.worker;

// Worker.builder() function
// Reads case context: recoveryVerification (status, verifiedNodes, divergedNodes, convergenceTime)
// Reads case context: containmentExecution (executionTimestamp) for DORA timeline
// Produces WorkerResult with RecoveryVerificationResultOutput
```

**Module:** `app/`
**Package:** `io.casehub.soc.worker`

#### SocHumanNodeHandler

Bridges desiredstate review nodes to WorkItems via WorkItemService.

```java
package io.casehub.soc.engine;

// @Alternative @ApplicationScoped
// implements HumanNodeHandler
// Displaces NoOpHumanNodeHandler
// onProvision(node, context):
//   1. Check idempotency: query WorkItemService for existing WorkItem keyed by node ID
//   2. If exists: return StepOutcome.Succeeded
//   3. If not: create WorkItem via WorkItemService
//      - scope: casehubio/soc/recovery-verification
//      - candidateGroups: [SOC_MANAGER]
//      - payload includes SocDivergenceReviewSpec context (faulted node, fault type, fault message)
//   4. Return StepOutcome.Succeeded
```

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

#### SocReviewNodeActualStateAdapter

ActualStateAdapter for review nodes — checks WorkItem completion status.

```java
package io.casehub.soc.engine;

// @ApplicationScoped
// implements ActualStateAdapter
// handledTypes() returns Set.of(SocDivergenceReviewSpec.REVIEW_TYPE)
// readActual():
//   - For each review node: query WorkItemService for completion status
//   - COMPLETED → PRESENT
//   - Not completed → ABSENT
// Closes the feedback loop: WorkItem completion → PRESENT → review node resolved
```

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

#### SocRecoveryLifecycleObserver

CDI observer that stops the reconciliation loop when the incident case closes.

```java
package io.casehub.soc.engine;

// @ApplicationScoped
// Observes CaseLifecycleEvent
// When case transitions to terminal state (COMPLETED, CANCELLED):
//   - Extract tenancyId = "case:" + caseId
//   - Call LifecycleManager.stop(tenancyId)
// Defensive: orphan sweeper scheduled task detects reconciliation loops
//   whose case is already completed, stops them
```

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

### SocStepType additions

| Value | Purpose |
|-------|---------|
| `RECOVERY_VERIFIED` | Per-node convergence confirmation with DORA timeline |
| `RECOVERY_DIVERGED` | Verification timeout — containment action did not take effect |

`SocLedgerEntryWriter.REQUIRED_METADATA` additions:
- `RECOVERY_VERIFIED`: `actionType`, `nodeId`, `convergenceTimestamp`, `executionToConvergenceMs`
- `RECOVERY_DIVERGED`: `actionType`, `nodeId`, `lastObservedStatus`, `timeoutMs`

### PII sanitisation

Node IDs embed operational identifiers (hostnames, IP addresses, credential IDs) — e.g., `soc:block-ip:ip-192.168.1.100`. IP addresses are personal data under GDPR (CJEU C-582/14). Extend `SocPiiSanitiser` to handle `RECOVERY_VERIFIED` and `RECOVERY_DIVERGED` entries: sanitise `nodeId` fields that embed IPs, hostnames, and credential identifiers before ledger write.

## Ledger Entry Chain Extension

Extends the existing containment ledger chain (from #40 spec):

### Converged verification (per node)
```
LedgerEntry: RECOVERY_VERIFIED
  actionType: ISOLATE_HOST
  nodeId: soc:isolate-host:host-abc123
  convergenceTimestamp: 2026-09-11T10:16:00Z
  executionToConvergenceMs: 57000
  causedByEntryId: <containment-executed-entry-id>
```

### Diverged verification (timeout)
```
LedgerEntry: RECOVERY_DIVERGED
  actionType: BLOCK_IP
  nodeId: soc:block-ip:ip-192.168.1.100
  lastObservedStatus: ABSENT
  timeoutMs: 1800000
  causedByEntryId: <containment-executed-entry-id>
```

## Verification Timeout (D12)

Severity-dependent, configurable via tenant preferences:

| Severity | Timeout | Rationale |
|----------|---------|-----------|
| P1 (CRITICAL) | 30 minutes | Must not block case longer than P1 SLA window |
| P2 (HIGH) | 2 hours | Aligns with P2 completion SLA |
| P3+ | 24 hours | Lower-priority incidents allow time for transient resolution |

Post-timeout: reconciliation loop continues running. Timeout only unblocks the case plan — ThresholdFaultPolicy escalation and SOC manager WorkItems remain active until case closure (D1 lifecycle cleanup).

## Maven Dependency Additions

Add to SOC `api/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate-api</artifactId>
</dependency>
```

Add to SOC `app/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-desiredstate</artifactId>
</dependency>
```

The `api/` dependency is required because `SocContainmentNodeTypes`, `SocContainmentNodeSpec`, and `SocDivergenceReviewSpec` extend `NodeType`, `NodeSpec`, and `HumanGating` from the desiredstate API.

The runtime module brings: `ReconciliationLoop`, `LifecycleManager`, `TransitionPlanner`, `SimpleTransitionExecutor`, `FaultPolicyEngine`, `CdiNodeProvisionerRouter`, `DefaultActualStateAdapterRouter`, `NoOpHumanNodeHandler` (`@DefaultBean`), `NoOpConfigurationAdapter`.

## SocInvestigationCaseDescriptor Changes

Register two new workers:
- `recovery-verification-start` → `RuleRecoveryVerificationWorker`
- `recovery-verification-result` → `SocRecoveryVerificationResultWorker`

## SocCaseHub.augment() Changes

Register both new workers alongside the existing workers.

## Upstream Change Required

### RuleContainmentExecutionWorker — include actionParameters in output

The execution worker's output map (lines 67-76) omits `actionParameters` — the map is used to call the executor (line 56) but never placed in the output. Recovery verification needs these parameters to construct NodeSpec records with domain-specific verification targets (hostname, IP, credential ID).

Add to the output map:
```java
output.put("actionParameters", actionParams);
```

Without this change, the recovery verification worker cannot construct meaningful NodeSpec records from the case context. This is a prerequisite for this epic.

## Existing Code — No Changes Required

- `ContainmentExecutor` SPI — execution interface unchanged
- `LoggingContainmentExecutor` — default executor unchanged
- `RuleContainmentExecutionWorker` — execution worker unchanged
- `SocContainmentLedgerObserver` — existing containment ledger entries unchanged
- `SocActionRiskClassifier` — risk classification unchanged
- `ContainmentDecisionMatrix` — decision matrix unchanged
- `SocSlaBreachPolicy` — SLA escalation unchanged (operates at case level, not verification level)

## Testing Strategy

- **Unit tests:** Recovery verification worker (graph construction from execution output), verification listener (convergence/divergence/timeout detection), fault policy (escalation thresholds), human node handler (idempotent WorkItem creation), review node adapter (WorkItem status mapping)
- **Integration test** (`RecoveryVerificationIntegrationTest`): inject alert → full pipeline through containment → verify reconciliation starts → simulate convergence → verify RECOVERY_VERIFIED ledger entry → verify case context updated
- **Four verification scenarios:** converged (all PRESENT), diverged (timeout with ABSENT), partially verified (mixed), no-containment (binding not triggered)
- **Divergence escalation test:** simulate DRIFTED status → verify ThresholdFaultPolicy creates review node → verify HumanNodeHandler creates WorkItem → simulate WorkItem completion → verify review node resolves
- **DORA timeline verification:** verify `executionToConvergenceMs` is calculated from containment execution timestamp to convergence detection
- **Lifecycle cleanup test:** close case → verify reconciliation loop stops for that tenancyId
- **Non-converged test paths:** Use a test-scoped `@Alternative` ActualStateAdapter that returns configurable statuses per node (ABSENT, DRIFTED, UNKNOWN) to exercise diverged and partially-verified paths without external dependencies

## References

- `RuleContainmentExecutionWorker.java` — containment execution worker (upstream in pipeline)
- `ContainmentExecutor.java` / `ContainmentResult.java` — execution SPI
- `SocContainmentLedgerObserver.java` — existing containment ledger observer pattern
- `SocLedgerEntryWriter.java` — Merkle chain ledger writer
- `SocPiiSanitiser.java` — PII sanitisation for ledger entries
- `incident-investigation.yaml` — case plan definition
- `SocInvestigationCaseDescriptor.java` — worker registration
- `SocCaseHub.java` — case hub augmentation
- `SocActionType.java` — action types with gate policies
- `SocSlaBreachPolicy.java` — SLA timing reference
- `CaseHubRuntime.signal()` (engine api/ line 60) — case context mutation API
- `DesiredStateGraph` (desiredstate api/) — graph interface
- `ActualStateAdapter` (desiredstate api/) — actual state reading SPI
- `GlobalReconciliationListener` (desiredstate api/) — per-cycle callback
- `ThresholdFaultPolicy` (desiredstate api/) — fault escalation
- `HumanNodeHandler` (desiredstate api/) — human-gated node bridge
- `NodeProvisioner` (desiredstate api/) — provisioning SPI
- `LifecycleManager` (desiredstate runtime/) — lifecycle orchestration
- `ReconciliationLoop` (desiredstate runtime/) — reconciliation engine
- `SimpleTransitionExecutor` (desiredstate runtime/) — transition execution
- `NoOpHumanNodeHandler` (desiredstate runtime/) — default human handler
- `OrgUnitNodeSpec` (eidos org-runtime/) — platform NodeSpec pattern reference
- Containment execution spec (#40) — upstream pipeline design
- DOMAIN.md §Phase 3 — recovery verification requirement
- AGENTIC-HARNESS-GUIDE.md — harness application conventions
