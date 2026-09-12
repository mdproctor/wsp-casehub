# Decisions — Desired-State Recovery Verification (#49)

## D1: Per-case reconciliation scope

**Choice:** One DesiredStateGraph per incident case, created after containment executes. Nodes represent the expected post-containment state for that incident. Graph lifecycle tied to the case — stopped when the incident closes via a `CaseLifecycleEvent` CDI observer that calls `LifecycleManager.stop()`. A scheduled orphan sweeper detects reconciliation loops whose case is already completed (defensive against observer failure) and stops them.
**Tenancy convention:** `tenancyId` is `case:<caseId>` — a scoping key for the reconciliation loop, not an organizational tenant. The desiredstate API's `tenancyId` parameter is a generic partition key; the eidos org module uses it for organizational boundaries, SOC uses it for per-incident boundaries.
**Alternatives:**
- Shared tenant graph — single long-lived graph per SOC tenant tracking all active containment states via `DesiredStateGraph.overlay()` with node ID prefix `case:<uuid>:`. More efficient (one loop) but: (a) one slow adapter blocks all incidents, (b) `CompletionCondition.allPresent()` checks all incidents together, (c) `FaultPolicy` fires per-tenant so a fault in one incident affects others, (d) lifecycle management loses per-case stop/start semantics, (e) DORA per-incident timelines require parsing node ID prefixes — fragile.
- Per-action graph — one graph per individual containment action. Most granular but loses cross-action dependency modelling within an incident.
**Rationale:** Clean per-incident DORA timeline (execution → convergence), natural lifecycle boundary (case close stops reconciliation), simple reasoning about convergence per incident. The `case:<caseId>` convention makes the scoping explicit and avoids conflating with organizational tenancy. `ReconciliationStateStore.remove(tenancyId)` is called by `TenantLoop.stop()`, ensuring cleanup on case closure.
**Trade-offs:** Multiple concurrent incidents mean multiple reconciliation loops. Acceptable — desiredstate's per-tenant `TenantLoop` in `ConcurrentHashMap` handles this natively.
**Sources:** `DesiredStateGraph` (api/), `LifecycleManager` (runtime/), `ReconciliationLoop.TenantLoop` (runtime/), `CaseLifecycleEvent` (engine), DOMAIN.md Phase 3
**Exploration:** quick
**Status:** revised — added lifecycle cleanup mechanism (CaseLifecycleEvent observer + orphan sweeper) and explicit tenancy convention (R1-01, R1-02)

## D2: Two case plan bindings for recovery verification

**Choice:** Add two bindings in `incident-investigation.yaml`:
1. `recovery-verification-start` — fires when `containmentExecution != null AND containmentExecution.executed == true AND recoveryVerification == null`. Worker reads execution output, builds DesiredStateGraph, starts LifecycleManager, writes `recoveryVerification: { status: "started", startedAt: <timestamp> }` to case context, and returns.
2. `recovery-verification-result` — fires when `recoveryVerification.status != "started"` (i.e. the GlobalReconciliationListener has updated context with verified/diverged). Records DORA timeline entry.

The `executed == true` guard prevents the binding from firing for rejected or no-containment cases, avoiding spurious empty-graph verification.
**Alternatives:**
- CDI observer on execution event — more decoupled but recovery step invisible in case plan, bad for audit trail and DORA compliance.
- Single binding that blocks until convergence — rejected for blocking the worker thread pool during real EDR verification.
- Single binding only — insufficient because the async verification result must be captured by a second binding.
**Rationale:** Two bindings reflect the async nature: start (synchronous, fast) and result (triggered by async convergence detection). Case plan YAML remains the authoritative lifecycle record — both steps visible for SOC2/DORA audit. The `executed == true` guard prevents spurious verification when containment was rejected or not recommended.
**Trade-offs:** Two bindings add complexity to the case plan. Acceptable — the bindings reflect the real lifecycle and the complexity is the compliance requirement.
**Sources:** `incident-investigation.yaml`, containment execution spec (#40) §Pipeline Flow, `ContainmentExecutionOutput`
**Exploration:** quick
**Status:** revised — split into two bindings with explicit guard for executed containment (R1-03, R1-04)

## D3: Default ActualStateAdapter mirrors the executor

**Choice:** `LoggingActualStateAdapter` returns PRESENT when the executor logged success. This is a dev/test-only default — semantically, it asserts verification without checking, which is acceptable in development but invalid in production. Real adapters (EDR API, firewall API) swap in via CDI `@Alternative`. A startup health check warns when no real adapter is configured in non-dev profiles.
**Alternatives:**
- Always UNKNOWN — more semantically honest but causes the reconciliation loop to attempt re-provisioning every cycle (the planner creates PROVISION steps for UNKNOWN nodes), which feeds into the NodeProvisioner concern (D8). The loop never converges, making dev/test unusable without additional workarounds.
- Configurable simulation — enables demo divergence scenarios but adds complexity.
**Rationale:** Follows the CDI displacement pattern (`@DefaultBean` mocks, `@Alternative` overrides). PRESENT is the only status that lets the default pipeline converge end-to-end in dev/test. The semantic dishonesty is acknowledged and mitigated by: (a) startup health check in non-dev profiles, (b) documentation that this adapter must be displaced for production.
**Trade-offs:** Default adapter cannot detect real divergence. Risk of reaching production mitigated by startup health check, not by making the default dev pipeline non-functional.
**Sources:** `LoggingContainmentExecutor.java`, `NodeStatus` enum (api/), `CompletionCondition.allPresent()` (api/)
**Exploration:** quick
**Depends on:** D1 (per-case scope determines the reconciliation context), D6 (NodeTypes determine which adapter handles which action types via `handledTypes()`)
**Status:** revised — acknowledged semantic concern, added production readiness gate, fixed dependency from D2 to D1/D6 (R1-05, R1-15)

## D4: ThresholdFaultPolicy escalation on divergence

**Choice:** Use desiredstate's `ThresholdFaultPolicy` configured with:
- `faultTypes`: `NODE_DEGRADED` (containment drifted — DRIFTED status) and `PROVISION_FAILED` (re-verification failed — provisioner returned Failed)
- `nodeTypes`: SOC containment types (`soc:isolate-host`, `soc:block-ip`, etc.)
- Tier 1: threshold 1 — retry verification (poll actual state on next cycle)
- Tier 2: threshold 2 — escalate via `addReviewNode(ReviewSpecFactory)` creating a human-gated review node
- SOC's `HumanNodeHandler` implementation bridges review nodes to `WorkItemService`, creating a WorkItem in the SOC manager's queue (see D11).
**Alternatives:**
- Immediate escalation — any divergence immediately creates a WorkItem. Simpler but may cause false escalations from transient network delays during the first check after containment executes.
- Higher threshold (escalate at 3) — allows more retries but 15+ minutes of unverified containment is unacceptable for critical security incidents.
**Rationale:** For security containment, one retry cycle is sufficient grace period for the action to take effect. One failed re-check (threshold 2) should escalate — 10 minutes of unverified containment is the maximum acceptable delay. NODE_DEGRADED fires when `detectDrift()` finds DRIFTED status; PROVISION_FAILED fires when the no-op SOC provisioner fails (D8).
**Trade-offs:** Tight thresholds may cause noise for transient adapter errors. Mitigated by the retry tier (threshold 1) absorbing the first transient failure.
**Sources:** `ThresholdFaultPolicy` (desiredstate api/), `FaultType` enum (api/), `FaultPolicy.addReviewNode()`, `ReconciliationLoop.detectDrift()` (runtime/)
**Exploration:** quick
**Status:** revised — reduced threshold from 3 to 2, specified FaultTypes, clarified review node to WorkItem bridge (R1-06, R1-07)

## D5: GlobalReconciliationListener for per-node convergence recording

**Choice:** Implement `GlobalReconciliationListener` that tracks per-node convergence rather than whole-graph convergence. On each cycle, for each node transitioning to PRESENT: write a RECOVERY_VERIFIED ledger entry with `causedByEntryId` linking to the execution entry, record execution→convergence time for DORA timeline. When ALL nodes converge: signal the case plan via `CaseHubRuntime.signal()` with `recoveryVerification: { status: "verified", convergenceTime: ... }`.
**Role separation:** The LifecycleManager handles lifecycle phase management (phase transitions, lifecycle removal via `CompletionCondition`). The GlobalReconciliationListener handles SOC-specific reactions (per-node ledger writes, case context updates, DORA recording). These are complementary, not redundant — LifecycleManager's check serves lifecycle semantics, the listener serves SOC compliance.
**Alternatives:**
- LifecycleManager CDI event on phase completion — architecturally cleaner (no per-cycle overhead, no filtering), but requires a platform change to add CDI events to `LifecycleManager.onCycleCompleted()`. Filed as desirable future enhancement; current design works with existing API.
- Whole-graph CompletionCondition.allPresent() only — loses per-node DORA timelines. When multiple containment actions are in play (isolate host + block IP), a successful action's DORA timeline should not be delayed by a failed action.
- Worker polls ActualStateAdapter directly — loses fault policy, event sourcing, and all reconciliation infrastructure.
**Rationale:** Per-node convergence gives granular DORA timelines and lets the case plan react to partial verification. The GlobalReconciliationListener fires every cycle for all tenants but the filter (check node type prefix `soc:`) is lightweight.
**Trade-offs:** Per-cycle listener invocation for all tenants. Acceptable — the filter is O(1) for non-SOC tenants.
**Sources:** `GlobalReconciliationListener` (desiredstate api/), `ReconciliationLoop.fireGlobalListeners()` (runtime/), `LifecycleManager.onCycleCompleted()` (runtime/)
**Exploration:** quick
**Depends on:** D1 (per-case scope means the listener checks a specific incident's graph)
**Status:** revised — per-node convergence instead of whole-graph, clarified role separation with LifecycleManager (R1-08, R1-09)

## D6: One NodeType per containment action category with domain-specific NodeSpec records

**Choice:** `NodeType.of("soc:isolate-host")`, `NodeType.of("soc:block-ip")`, etc. Each gets its own `NodeSpec` record implementation carrying action parameters from the containment execution output:
- `IsolateHostSpec(NodeType type, String hostname, String edpAgentId)` — ActualStateAdapter queries EDR API with the edpAgentId to verify isolation status
- `BlockIpSpec(NodeType type, String ipAddress, String firewallRuleId)` — ActualStateAdapter queries firewall API with the ruleId to verify block
- `RevokeCredentialsSpec(NodeType type, String credentialId, String iamProvider)` — ActualStateAdapter queries IAM API with credentialId

The recovery verification worker maps `ContainmentExecutionOutput` (action type + parameters) to the appropriate NodeSpec implementation when building the DesiredStateGraph.
**Alternatives:**
- Single generic NodeType — `NodeType.of("soc:containment-action")` for all. Simpler but loses per-type `ActualStateAdapter` routing (`handledTypes()`) and per-type `NodeProvisioner` routing.
- NodeSpec without domain parameters — minimal interface (`nodeType()` + `humanGating()`) with no way to carry verification targets. `ActualStateAdapter.readActual()` receives the full graph but has no parameters to query the external system.
**Rationale:** Enables per-type ActualStateAdapter routing (different verification logic for host isolation vs IP blocking vs credential revocation). Follows the platform pattern: `OrgUnitNodeSpec(OrganizationalUnit)` and `OrgRelationshipNodeSpec(AgentRelationship)` carry domain data as record fields.
**Trade-offs:** More NodeType constants and NodeSpec implementations. Maps 1:1 to `SocActionType` which already exists and is stable.
**Sources:** `SocActionType.java`, `ActualStateAdapter.handledTypes()`, `OrgUnitNodeSpec` (eidos org-runtime/), `OrgRelationshipNodeSpec` (eidos org-runtime/)
**Exploration:** quick
**Status:** revised — added concrete NodeSpec implementations showing parameter propagation from containment output (R1-10)

## D7: Worker starts lifecycle, listener signals case context via CaseHubRuntime

**Choice:** Recovery verification worker builds the DesiredStateGraph, starts a LifecycleManager with a single verification phase, and returns a preliminary result (`verification_started`). GlobalReconciliationListener detects convergence/divergence and updates case context via `CaseHubRuntime.signal(caseId, "recoveryVerification", { status: "verified", convergenceTime: ... })`. The engine applies the context change, fires `CONTEXT_CHANGED`, and the `recovery-verification-result` binding evaluates its `when` guard. Decoupled, non-blocking.
**Alternatives:**
- Worker polls reconciliation synchronously — simpler but blocks the worker thread pool for minutes during real EDR verification.
- Fire-and-forget with timeout binding — most decoupled but requires timer support in case plan YAML.
**Rationale:** Reconciliation is inherently async (polling external systems). `CaseHubRuntime.signal()` is the correct API for updating case context from outside the engine — it applies the update, writes the event log, and dispatches `CONTEXT_CHANGED`. `CaseLifecycleEvent` is a notification event fired BY the engine on lifecycle transitions, not a mechanism for external code to write context.
**Trade-offs:** Two-phase result reporting (started → verified/diverged) adds a case plan binding. Acceptable — the case plan should reflect the async nature of verification.
**Sources:** `CaseHubRuntime.signal()` (engine api/), `CaseLifecycleEvent` (engine — notification only), `GlobalReconciliationListener`, `LifecycleManager` (desiredstate runtime/)
**Exploration:** quick
**Depends on:** D2 (case plan bindings), D5 (listener writes convergence)
**Status:** revised — corrected mechanism from CaseLifecycleEvent (notification) to CaseHubRuntime.signal() (mutation) (R1-11)

## D8: No auto-reprovisioning — SOC NodeProvisioner returns Failed

**Choice:** SOC `NodeProvisioner` implementations for containment types (`soc:isolate-host`, `soc:block-ip`, etc.) return `ProvisionResult.Failed("re-provisioning disabled — containment divergence requires human review")`. They do NOT re-execute containment actions. The `Failed` result maps to `StepOutcome.Failed` in the `TransitionExecutor`, which triggers `FaultEvent(PROVISION_FAILED)` in the `ReconciliationLoop.faultFeedback()` method, feeding into `ThresholdFaultPolicy` escalation (D4).
**Alternatives:**
- Auto-reprovision — provisioner re-executes the containment action. Dangerous: a transient adapter error triggering `WIPE_ENDPOINT` is catastrophic. An operator who deliberately lifted isolation would have the loop re-isolate automatically, overriding human judgment.
- No-op provisioner (return Success/Skipped) — the loop silently accepts divergence with no escalation. `faultFeedback()` only processes `Failed` and `Rejected` outcomes — `Skipped` and `Succeeded` generate no fault events.
- HumanGating.ALL on all containment nodes — forces human approval for initial provisioning too (not just re-provisioning), blocking the initial verification flow.
**Rationale:** SOC recovery verification is a detection system, not a remediation system. The reconciliation loop detects divergence; humans decide whether to re-execute. The `Failed` provisioner result is the only path that generates a `FaultEvent` without actually performing a dangerous action.
**Trade-offs:** The loop attempts provisioning every cycle for ABSENT nodes, and the provisioner fails every time, incrementing the fault count. This is the correct model: the fault count drives escalation via ThresholdFaultPolicy (D4).
**Sources:** `NodeProvisioner.provision()` (api/), `ProvisionResult.Failed` (api/), `StepOutcome.Failed` (api/), `ReconciliationLoop.faultFeedback()` (runtime/)
**Exploration:** quick (surfaced by review)
**Status:** captured — implicit decision surfaced by R1-12

## D9: casehub-desiredstate-runtime dependency

**Choice:** Add `casehub-desiredstate-runtime` as a dependency of the SOC project. The runtime module provides `ReconciliationLoop`, `LifecycleManager`, `TransitionPlanner`, `TransitionExecutor`, `FaultPolicyEngine`, `CdiNodeProvisionerRouter`, `DefaultActualStateAdapterRouter`, `NoOpHumanNodeHandler` (`@DefaultBean`), and `NoOpConfigurationAdapter`.
**Alternatives:**
- API-only dependency with custom reconciliation loop — avoids bringing the runtime's CDI beans and default implementations, but duplicates core reconciliation infrastructure.
**Rationale:** The runtime module IS the reconciliation infrastructure. All seven original decisions (D1–D7) reference runtime classes (`LifecycleManager`, `ReconciliationLoop`). The runtime's `@DefaultBean` implementations (`NoOpHumanNodeHandler`, `NoOpConfigurationAdapter`) are safe defaults that SOC overrides via `@Alternative`. `CbrFaultPolicy` and `CbrSituationRecompiler` (present in runtime) are not activated without CBR configuration — no conflict with existing SOC integration patterns.
**Trade-offs:** Larger dependency footprint. Acceptable — the module is purpose-built for this use case.
**Sources:** `casehub-desiredstate-runtime` module (desiredstate/runtime/), SOC pom.xml (currently has api only)
**Exploration:** quick (surfaced by review)
**Status:** captured — implicit decision surfaced by R1-13

## D10: Direct graph construction, no GoalCompiler

**Choice:** The recovery verification worker builds the `DesiredStateGraph` directly from `ContainmentExecutionOutput`, without using the `GoalCompiler<G>` pattern.
**Alternatives:**
- `SocRecoveryGoalCompiler implements GoalCompiler<ContainmentExecutionOutput>` — adds the canonical integration point. Would enable future composition of verification goals from multiple sources.
**Rationale:** The `GoalCompiler` pattern adds value when: (a) goals need to be compiled by a reusable component used by multiple callers, or (b) the compilation logic needs to vary by goal type. For SOC recovery verification, there is one caller (the recovery verification worker), one goal shape (verify that these containment actions took effect), and the mapping from execution output to graph is direct (one NodeSpec per executed action). A GoalCompiler adds indirection without architectural benefit. If SOC later needs to compose verification goals from multiple sources, the pattern can be introduced then — `CompilationResult.lifecycle()` and `CompilationResult.single()` are the entrypoints, and the refactoring is mechanical.
**Trade-offs:** Loses the `GoalCompiler` extension point. Acceptable — YAGNI applies when the extension has one potential caller and one goal shape.
**Sources:** `GoalCompiler<G>` (desiredstate api/), `OrgGoalCompiler` (eidos org-runtime/ — the canonical example), `CompilationResult` (api/)
**Exploration:** quick (surfaced by review)
**Status:** captured — implicit decision surfaced by R1-14

## D11: HumanNodeHandler bridges desiredstate review nodes to WorkItems

**Choice:** SOC implements `HumanNodeHandler` (displacing `NoOpHumanNodeHandler` via `@Alternative`) that creates a `WorkItem` via `WorkItemService` when a human-gated review node is provisioned. The handler returns `StepOutcome.Succeeded` and is idempotent — before creating a WorkItem, it checks whether one already exists for this review node (keyed by node ID). If the WorkItem already exists, it returns `Succeeded` without creating a duplicate.
**Return value rationale:** `StepOutcome.Succeeded` is the only viable option. `Failed` triggers `faultFeedback()` creating `FaultEvent(PROVISION_FAILED)`, but review node types are in `ThresholdFaultPolicy.ignoreTypes` (auto-populated from tier action `outputNodeType()`), so the fault is silently ignored — harmless but generates processing overhead every cycle. `Skipped` is equivalent to the no-op default. `Rejected` has wrong semantics.
**Re-invocation cycle:** `DefaultActualStateAdapterRouter` returns `NodeStatus.UNKNOWN` for uncovered node types (line 62-64). UNKNOWN causes the `TransitionPlanner` to create a PROVISION step (line 60-64: `UNKNOWN → provision = true`). The handler is re-invoked every cycle. Idempotency ensures no duplicate WorkItems.
**Reverse feedback path:** An `ActualStateAdapter` for review node types checks WorkItem completion status. When the SOC manager completes the WorkItem → adapter returns `PRESENT` → the planner stops creating PROVISION steps for the review node → the cycle quiesces. This adapter's `handledTypes()` returns the review node type(s) used by `ThresholdFaultPolicy.addReviewNode()`.
**Review node NodeSpec:** The `ReviewSpecFactory` creates a `SocDivergenceReviewSpec(NodeType reviewType, NodeId faultedNodeId, FaultEvent event)` carrying the faulted containment node ID, fault type, and fault message. The WorkItem includes this context so the SOC manager sees which containment action diverged and why.
**Alternatives:**
- Rely on NoOpHumanNodeHandler — review nodes are silently skipped (`StepOutcome.Skipped`), SOC manager never sees the escalation.
- Custom CDI event from FaultPolicy — bypasses the desiredstate human gating mechanism. Loses the review node's dependency semantics (the review node blocks the faulted node's resolution until a human acts).
- Remove review node from graph on WorkItem completion (via graph mutation in handler) — more complex and less idiomatic than an ActualStateAdapter returning PRESENT.
**Rationale:** The `HumanNodeHandler` is the platform's intended bridge between desiredstate's human gating and external work management. The eidos org module doesn't need this bridge (org structure is managed by admins, not WorkItems), but SOC does — divergence escalations must appear in the SOC manager's inbox. The `@Alternative` CDI displacement is the standard pattern. The ActualStateAdapter for review nodes closes the feedback loop: WorkItem completion → PRESENT status → review node resolved.
**Trade-offs:** The handler is re-invoked every cycle for unresolved review nodes (idempotent check + `Succeeded` return). Acceptable — the overhead is minimal compared to the adapter calls.
**Sources:** `HumanNodeHandler` (desiredstate api/), `NoOpHumanNodeHandler` (desiredstate runtime/), `SimpleTransitionExecutor.executeProvision()` lines 90-91 (runtime/), `DefaultActualStateAdapterRouter.readActual()` lines 62-64 (runtime/), `TransitionPlanner.plan()` lines 60-64 (runtime/), `ThresholdFaultPolicy` ignoreTypes auto-population (api/), `WorkItemService` (casehub-work/)
**Exploration:** quick (surfaced by review, refined by R2-01)
**Status:** revised — specified handler return value (Succeeded with idempotency), reverse feedback path (ActualStateAdapter for review nodes), and re-invocation cycle (R2-01)

## D12: Verification timeout — case plan unblocks on permanent divergence

**Choice:** The `GlobalReconciliationListener` tracks elapsed time since `recoveryVerification.startedAt` on each reconciliation cycle. When the configurable timeout expires and not all nodes have converged, the listener signals a terminal status via `CaseHubRuntime.signal()`:
- If some nodes converged: `recoveryVerification: { status: "partially_verified", verifiedNodes: [...], divergedNodes: [...], elapsedTime: ... }`
- If no nodes converged: `recoveryVerification: { status: "diverged", divergedNodes: [...], elapsedTime: ... }`

This fires the `recovery-verification-result` binding (its `when` guard checks `status != "started"`), unblocking the case plan.

**Timeout values** (severity-dependent, configurable via tenant preferences):
- P1 (CRITICAL): 30 minutes — containment verification for critical incidents cannot block the case longer than the P1 SLA claim window
- P2 (HIGH): 2 hours — aligns with the P2 completion SLA
- P3+: 24 hours — lower-priority incidents allow more time for transient issues to resolve

**Post-timeout behavior:** The reconciliation loop continues running after the timeout fires. The timeout only unblocks the case plan — it does not stop the loop. The loop keeps detecting divergence, the ThresholdFaultPolicy keeps escalating (D4), and the SOC manager's WorkItem (D11) remains active. The D1 lifecycle cleanup stops the loop when the case eventually closes.
**Per-node DORA compliance:** Per-node RECOVERY_VERIFIED ledger entries (D5) are already recorded as each node converges. The timeout does not affect these — successfully verified actions have their DORA timelines recorded regardless of whether other actions time out.
**Alternatives:**
- No timeout — case plan stalls indefinitely on permanent divergence. A single unreachable external system (IAM down for maintenance) blocks the entire incident lifecycle.
- Timer binding in case plan YAML — rejected in D7 for lack of timer support in the case plan.
- LifecycleManager phase timeout — requires a platform change (Phase record has CompletionCondition but no timeout). Desirable future enhancement but not available today.
**Rationale:** The GlobalReconciliationListener already fires every cycle and has access to both convergence state and elapsed time. Adding timeout detection is a natural extension of its convergence check: (a) all PRESENT → verified, (b) timeout expired → diverged/partially_verified, (c) neither → continue. The severity-dependent timeout aligns with the existing SLA structure (ARC42STORIES.MD Layer 3).
**Trade-offs:** Timeout values are heuristics — too short creates false diverged signals for slow external systems, too long leaves the case plan blocked. Configurable via tenant preferences to allow tuning per deployment.
**Sources:** `GlobalReconciliationListener` (desiredstate api/), `CaseHubRuntime.signal()` (engine api/), `SocSlaBreachPolicy` (soc/ — SLA timing reference), ARC42STORIES.MD Layer 3 SLA tables
**Exploration:** quick (surfaced by review)
**Depends on:** D2 (recovery-verification-result binding guard), D5 (per-node convergence), D7 (CaseHubRuntime.signal mechanism)
**Status:** captured — implicit decision surfaced by R2-02
