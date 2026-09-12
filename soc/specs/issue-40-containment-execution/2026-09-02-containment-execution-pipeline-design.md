# Containment Execution Pipeline — Design Spec

**Epic:** #40 — Containment execution pipeline — gated response with compliance audit
**Issues:** #41 (approval gate), #42 (execution worker), #43 (compliance audit), #44 (case plan bindings), #45 (e2e test)
**Date:** 2026-09-02
**Revision:** 2 — incorporates light design review findings (R1-02 through R1-11)

---

## Problem

The current case plan ends at `containment-recommendation → analyst-review`. The containment recommendation worker produces a `PlannedAction` with action type, risk score, and parameters, but nothing acts on it after the analyst review. The `analyst-review` humanTask conflates triage review ("is this a real incident?") with implicit containment approval ("should we act?").

Without a gated execution pipeline:
- GDPR Art. 22 decision attribution has no teeth — irreversible actions lack a traceable approval chain
- DORA response timelines are not measurable — no timestamp from classification to containment
- SOC2 audit evidence is incomplete — no tamper-evident record of what was executed and who approved it

## Approach

Extend the case plan with a containment-execution binding that fires after the engine's built-in `PlannedAction → ActionGate` lifecycle handles gating and approval. The engine already handles `PlannedAction → ActionRiskClassifier → GateRequired → WorkItem creation → approval/rejection → context signals` — we observe those signals, execute the containment, and write audit entries.

### Key insight from review (R1-02)

The engine has a complete ActionGate lifecycle:
1. Worker produces `PlannedAction` → engine runs `ChainedActionRiskClassifier` → if `GateRequired`, engine pauses the case, creates a WorkItem via `JudgmentScheduler` / `ActionGateWorkItemHandler`
2. WorkItem COMPLETED → `ActionGateApprovedHandler` writes `actionGateApproved` (with `actionType`, `approvedBy`, `resolution`) to case context, re-fires `CONTEXT_CHANGED`
3. WorkItem REJECTED → `ActionGateRejectedHandler` writes `actionGateRejected` to case context, fires `CONTEXT_CHANGED`
4. For `Autonomous` actions → `PlannedAction` output applied immediately, recommendation binding completes normally

We DO NOT need custom gate workers, approval humanTasks, or approval handlers. The engine handles all of this.

## Pipeline Flow

```
cbr-retrieval → ioc-enrichment → attck-mapping → containment-recommendation
    → analyst-review              (triage: "is this real?" — BEFORE containment)
    → containment-execution        (execute via ContainmentExecutor SPI)
```

The engine's ActionGate lifecycle runs transparently between the recommendation worker producing a `PlannedAction` and the case context being updated. When the recommendation worker emits a `PlannedAction`:

- **Autonomous (low-risk):** Engine applies output immediately. Recommendation binding completes. Triage review follows, then execution.
- **Gated (high-risk):** Engine pauses the case, creates a WorkItem on the oversight channel. SOC manager approves/rejects. Engine writes `actionGateApproved` or `actionGateRejected` to context. Recommendation binding completes (or is rejected). Triage review follows, then execution (if approved).
- **No containment (LOW severity):** `ContainmentDecisionMatrix` returns no action. Recommendation worker completes with no `PlannedAction`. No gate. Triage review fires. Execution binding skips (nothing to execute).

### Ordering rationale (R1-04)

Triage review (`analyst-review`) comes BEFORE containment execution. This ensures:
- False positives don't receive containment actions
- The analyst confirms the threat before irreversible actions occur
- For the gated path, the SOC manager approved the *specific action* via the ActionGate WorkItem, but the Tier-1 analyst still confirms the *incident* is real

### Three execution paths

**Gated + approved:** recommendation → engine gates → SOC manager approves via WorkItem → analyst reviews incident → execution → 3 ledger entries

**Autonomous:** recommendation → engine applies immediately → analyst reviews incident → execution → 2 ledger entries

**Rejected:** recommendation → engine gates → SOC manager rejects → analyst reviews incident → no execution → 2 ledger entries (gate + rejection)

**No containment:** recommendation (no PlannedAction) → analyst reviews incident → execution binding skips → 1 ledger entry (triage only)

## New Components

### ContainmentExecutor (SPI)

Interface in the `api/` module for pluggable containment execution backends.

```java
public interface ContainmentExecutor {
    ContainmentResult execute(String actionType, Map<String, Object> parameters,
                              ContainmentContext context);
}
```

- `ContainmentResult`: record — `success` (boolean), `timestamp` (Instant), `details` (String), `errorReason` (String, nullable), `retryable` (boolean)
- `ContainmentContext`: record — `caseId` (UUID), `incidentId` (String), `approver` (String, nullable for autonomous), `tenancyId` (String), `timeoutMs` (long, default 30_000)

Contract-level timeout: implementations MUST respect `timeoutMs` and return a failure result (with `retryable=true`) if the external system does not respond within the timeout. The worker handles retry decisions — the executor reports whether a failure is retryable.

Real implementations (EDR API, firewall API, IAM) swap in via CDI `@Alternative`.

**Module:** `api/`
**Package:** `io.casehub.soc.engine.spi`

### LoggingContainmentExecutor

Default `@DefaultBean` implementation. Logs the action at INFO level, records success, emits a CDI `ContainmentExecutedEvent`. No actual system integration.

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

### RuleContainmentExecutionWorker

Worker bound to the `containment-execution` capability. Fires after analyst-review completes (triage confirmed):

- Reads case context for `actionGateApproved` (gated path) or checks if `PlannedAction` was applied directly (autonomous path)
- If `actionGateRejected` is present → skip execution, produce result with `executed=false`
- If no `PlannedAction` was ever produced (no containment needed) → skip, produce result with `executed=false`
- Otherwise: delegates to injected `ContainmentExecutor`
- Produces `WorkerResult` with `ContainmentExecutionOutput`

**Module:** `app/`
**Package:** `io.casehub.soc.worker`

### ContainmentExecutionOutput

Worker output contract in `api/`.

```java
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

**Module:** `api/`
**Package:** `io.casehub.soc.worker.contract`

### SocContainmentLedgerObserver

Writes ledger entries for the containment decision chain. Three distinct triggers:

1. **Gate decision entry:** Observes `CaseLifecycleEvent` filtering for `actionGateApproved` or `actionGateRejected` context signals (written by engine's `ActionGateApprovedHandler` / `ActionGateRejectedHandler`). For autonomous actions, observes the containment-recommendation binding completion with a `PlannedAction` present.

2. **Approval/rejection entry:** Same `CaseLifecycleEvent` observer as (1) — the `actionGateApproved` / `actionGateRejected` context signals carry the approver identity, resolution text, and action type. Gate decision and approval are written from the same event but as separate ledger entries with `causedByEntryId` linking them.

3. **Execution result entry:** Observes `CaseLifecycleEvent` filtering for `containmentExecution` context key (set by `RuleContainmentExecutionWorker`). Writes execution result, DORA timeline fields.

Uses `SocPiiSanitiser` on action parameters before ledger write. All entries link via `causedByEntryId`.

**Module:** `app/`
**Package:** `io.casehub.soc.engine.compliance`

### SocStepType additions

Add to the existing `SocStepType` enum:

| Value | Purpose | Replaces |
|-------|---------|----------|
| `CONTAINMENT_GATE_DECISION` | Risk classification result (autonomous/gated) | New |
| `CONTAINMENT_APPROVAL` | Human approval with approver identity | Extends existing `CONTAINMENT_DECISION` |
| `CONTAINMENT_REJECTION` | Human rejection with reason | New |

Existing values `CONTAINMENT_DECISION` and `CONTAINMENT_EXECUTED` remain — `CONTAINMENT_DECISION` maps to the recommendation step (what was recommended), while `CONTAINMENT_APPROVAL` maps to the approval step (what was approved).

`SocLedgerEntryWriter.REQUIRED_METADATA` additions:
- `CONTAINMENT_GATE_DECISION`: `actionType`, `riskScore`, `gateDecision`, `confidenceScore`
- `CONTAINMENT_APPROVAL`: `actionType`, `approverId`, `resolution`
- `CONTAINMENT_REJECTION`: `actionType`, `rejectorId`, `rejectionReason`

## Case Plan YAML Changes

Updated `incident-investigation.yaml` using the actual binding syntax:

```yaml
- name: cbr-retrieval
  on: { contextChange: {} }
  when: ".cbrRetrieval == null"
  capability: cbr-retrieval

- name: ioc-enrichment
  on: { contextChange: {} }
  when: ".cbrRetrieval != null and .iocEnrichment == null"
  capability: ioc-enrichment

- name: attck-mapping
  on: { contextChange: {} }
  when: ".iocEnrichment != null and .attckMapping == null"
  capability: attck-mapping

- name: containment-recommendation
  on: { contextChange: {} }
  when: ".attckMapping != null and .containmentRecommendation == null"
  capability: containment-recommendation

# Triage review — analyst confirms this is a real incident BEFORE containment
- name: analyst-review
  on: { contextChange: {} }
  when: ".containmentRecommendation != null and .analystDecision == null"
  humanTask:
    scope: casehubio/soc/triage
    candidateGroups: [SOC_ANALYST]

# Containment execution — fires after triage confirms the incident
- name: containment-execution
  on: { contextChange: {} }
  when: ".analystDecision != null and .containmentExecution == null"
  capability: containment-execution
```

### Sequencing logic

- `containment-recommendation` fires after ATT&CK mapping. If its `PlannedAction` gets `GateRequired`, the engine pauses the case and creates a WorkItem — the binding only completes after the gate resolves.
- `analyst-review` fires after the recommendation completes (or after gate approval/rejection). Analyst confirms the incident is real.
- `containment-execution` fires after analyst confirms. The worker checks context for the gate outcome:
  - `actionGateApproved` present → execute the approved action
  - `actionGateRejected` present → skip execution, record rejection
  - No PlannedAction produced (LOW severity) → skip, record "no containment needed"
  - Autonomous (no gate) → execute the recommended action

### No-containment path (R1-03)

When `ContainmentDecisionMatrix` returns no recommendation (LOW severity), the recommendation worker returns `WorkerResult.of(output)` without a `PlannedAction`. No gate fires. The recommendation binding completes normally. `analyst-review` fires (`.containmentRecommendation != null` is satisfied because the output is written to context even without a PlannedAction). `containment-execution` fires and the worker detects there's nothing to execute — produces `ContainmentExecutionOutput` with `executed=false`.

## Ledger Entry Chain

### Gated containment (e.g., isolate host)

```
LedgerEntry #1: CONTAINMENT_GATE_DECISION
  actionType: ISOLATE_HOST
  riskScore: 0.95
  gateDecision: GATED
  candidateGroups: [SOC_MANAGER, CISO]
  causedByEntryId: <recommendation-entry-id>

LedgerEntry #2: CONTAINMENT_APPROVAL
  actionType: ISOLATE_HOST
  approverId: analyst-jane@corp.com
  approvalTimestamp: 2026-09-02T10:15:00Z
  resolution: "Host exhibits lateral movement indicators, approve isolation"
  causedByEntryId: <gate-decision-entry-id>

LedgerEntry #3: CONTAINMENT_EXECUTED
  actionType: ISOLATE_HOST
  executionResult: SUCCESS
  executionTimestamp: 2026-09-02T10:15:03Z
  detectionToContainmentMs: 45003
  executor: LoggingContainmentExecutor
  causedByEntryId: <approval-entry-id>
```

### Autonomous containment (e.g., block IP with low risk score)

```
LedgerEntry #1: CONTAINMENT_GATE_DECISION
  actionType: BLOCK_IP
  riskScore: 0.3
  gateDecision: AUTONOMOUS
  causedByEntryId: <recommendation-entry-id>

LedgerEntry #2: CONTAINMENT_EXECUTED
  actionType: BLOCK_IP
  executionResult: SUCCESS
  detectionToContainmentMs: 1200
  causedByEntryId: <gate-decision-entry-id>
```

### Rejected containment

```
LedgerEntry #1: CONTAINMENT_GATE_DECISION
  actionType: ISOLATE_HOST
  gateDecision: GATED
  causedByEntryId: <recommendation-entry-id>

LedgerEntry #2: CONTAINMENT_REJECTION
  actionType: ISOLATE_HOST
  rejectorId: analyst-bob@corp.com
  rejectionReason: "False positive — no lateral movement detected"
  causedByEntryId: <gate-decision-entry-id>
```

### No containment (LOW severity)

No containment-specific ledger entries. The recommendation step writes `CONTAINMENT_DECISION` with `recommendedAction=null`. Analyst review proceeds normally.

## Existing Code Changes

### RuleContainmentRecommendationWorker — fix (R1-08)

Add `confidenceScore` to `PlannedAction` parameters. Currently only includes `riskScore`, `severity`, `tactic`. Without `confidenceScore`, `ROTATE_API_KEY` (which uses `GatePolicy.CONFIDENCE_THRESHOLD`) always falls through to `missingContext()` → forced `GateRequired`.

### SocInvestigationCaseDescriptor — extend

Add `containment-execution` capability → `RuleContainmentExecutionWorker`.

### SocCaseHub.augment() — extend

Register the new worker alongside the existing 7.

## Existing Code — No Changes Required

- `SocActionRiskClassifier` — risk classification logic (engine calls it automatically)
- `ContainmentDecisionMatrix` — severity × tactic → action mapping
- `LlmContainmentRecommendationWorker` — LLM variant of recommendation
- `ContainmentRecommendationOutput` — recommendation output contract
- `SocSlaBreachPolicy` — escalation chain for SLA breaches
- `SocActionType` — action types with gate policies and candidate groups
- `SocEscalatedWorkItemHandler` — handles ESCALATED status (unrelated to containment approval)
- Engine ActionGate lifecycle — `ActionGateApprovedHandler`, `ActionGateRejectedHandler`, `ActionGateWorkItemHandler` (all working, no changes needed)

## Components Removed (vs. revision 1)

Per review finding R1-02, the following components from revision 1 are removed because they duplicate the engine's built-in ActionGate lifecycle:

- ~~`RuleContainmentGateWorker`~~ — engine handles gate creation/classification internally
- ~~`containment-approval` humanTask~~ — engine creates WorkItems automatically via `ActionGateWorkItemHandler`
- ~~`SocContainmentApprovalHandler`~~ — engine's `ActionGateApprovedHandler` / `ActionGateRejectedHandler` write context signals

## Worker Registration

`SocInvestigationCaseDescriptor` adds one new worker:
- `containment-execution` → `RuleContainmentExecutionWorker` (executes via SPI after triage confirmation)

`SocCaseHub.augment()` registers it alongside the existing 7 workers.

## Testing Strategy

- Unit tests: execution worker (all 4 paths), ledger observer (all entry types), containment executor SPI
- Integration test (`ContainmentPipelineIntegrationTest`): inject alert → full pipeline → verify ledger chain
- Four test scenarios: gated+approved, autonomous, gated+rejected, no-containment (LOW severity)
- Verify `causedByEntryId` chain integrity
- Verify DORA response timeline fields
- Verify `confidenceScore` fix — ROTATE_API_KEY with high confidence should be autonomous

## References

- `SocActionRiskClassifier.java:16` — risk classification implementation
- `SocEscalatedWorkItemHandler.java:14` — existing WorkItem observer pattern
- `SocIncidentLedgerObserver.java` — existing ledger observer pattern
- `SocResolutionLedgerObserver.java` — existing ledger observer pattern
- `SocLedgerEntryWriter.java` — Merkle chain mechanics
- `SocPiiSanitiser.java` — PII sanitisation for ledger entries
- `NoOpOversightGateService.java` — platform pattern for default SPI impls
- `incident-investigation.yaml` — current case plan definition
- `SocActionType.java` — action types with gate policies
- `SocInvestigationCaseDescriptor.java` — worker registration
- `ActionGateApprovedHandler.java:111-120` — engine writes `actionGateApproved` to context
- `ActionGateRejectedHandler.java:110-122` — engine writes `actionGateRejected` to context
- `ActionGateWorkItemHandler.java:52-68` — engine creates WorkItems from gate requests
- `ContainmentDecisionMatrix.java:18-21` — LOW severity returns null action
- `RuleContainmentRecommendationWorker.java:50-53` — PlannedAction parameters (needs confidenceScore fix)
- `io.casehub.api.spi.ActionRiskClassifier` — platform SPI interface
- `io.casehub.api.spi.RiskDecision` — Autonomous / GateRequired sealed types
- `io.casehub.worker.api.PlannedAction` — action record
- `io.casehub.work.api.WorkItemLifecycleEvent` — WorkItem CDI event
- `io.casehub.work.api.WorkItemStatus` — terminal statuses
- `docs/specs/slice-1-siem-critical-alert/2026-07-29-slice-1-design.md:486-496` — original containment approval flow design
