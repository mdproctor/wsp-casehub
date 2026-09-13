# Decisions — Containment Execution Pipeline (#40)

## D1: Separate triage review from containment approval

**Choice:** Split the current `analyst-review` humanTask into two distinct case plan steps: triage-review ("is this a real incident?") and containment-approval ("should we execute this action?").
**Alternatives:**
- Combined — keep single analyst-review for both concerns. Simpler flow but less granular audit.
- Conditional split — combined for autonomous, separate only when GateRequired. Reduces WorkItems but inconsistent audit trail.
**Rationale:** Separation gives distinct audit entries, independent SLA timers, and the ability to route triage to Tier-1 analysts while containment approval goes to SOC managers (via `SocActionType.candidateGroups()`). DORA requires demonstrable separation of detection and response decision-making.
**Trade-offs:** More WorkItems per incident, slightly longer mean time-to-containment for the gated path due to an extra approval step.
**Sources:** `SocEscalatedWorkItemHandler.java` (current conflation), `SocActionType.java` (candidate groups differ by action), `WorkItemLifecycleEvent` (CDI event model for observing outcomes)
**Exploration:** quick
**Status:** captured

## D2: Log-and-record default executor SPI

**Choice:** Default `ContainmentExecutor` implementation logs the action, records success to case context, and emits a CDI event. No actual system integration. Real EDR/firewall/IAM implementations swap in via CDI `@Alternative`.
**Alternatives:**
- Simulated with delay — adds configurable delay and failure probability. More demo-friendly but adds complexity.
- Dry-run flag — single impl with config flag. Forces explicit opt-in but conflates two responsibilities.
**Rationale:** Follows the existing CDI displacement pattern used throughout the platform (`@DefaultBean` mocks, `@Alternative` overrides). Keeps the SPI clean — the interface is what matters, not the default impl.
**Trade-offs:** Demo scenarios won't show realistic execution timing. Acceptable because the demo endpoint (`SocDemoResource`) can inject alerts that exercise the full pipeline without needing real containment.
**Sources:** `NoOpOversightGateService.java` (platform pattern for default SPI impls), `RuleContainmentRecommendationWorker.java` (existing worker pattern)
**Exploration:** quick
**Status:** captured

## D3: Three-phase audit trail (one ledger entry per phase)

**Choice:** Three distinct ledger entries per containment action: (1) gate decision — autonomous/gated + risk score, (2) approval outcome — approved/rejected + approver identity, (3) execution result — success/failure + timestamp. Each links via `causedByEntryId`.
**Alternatives:**
- Single composite entry — one entry capturing all phases. Simpler but harder to query and less granular for compliance.
- Two entries — gate+approval combined, execution separate. Moderate granularity but loses the ability to distinguish risk classification from human judgment.
**Rationale:** DORA requires demonstrable response timelines (detection → classification → decision → containment). SOC2 requires attribution of who approved what. Three entries map directly to these compliance dimensions. The `causedByEntryId` chain provides the causal narrative.
**Trade-offs:** More ledger writes per incident. Acceptable — the Merkle chain handles volume, and compliance auditors need the granularity.
**Sources:** `SocIncidentLedgerObserver.java`, `SocResolutionLedgerObserver.java` (existing patterns), `SocLedgerEntryWriter.java` (Merkle chain mechanics)
**Exploration:** quick
**Status:** captured

## D4: Single YAML path with conditional guards

**Choice:** One `containment-execution` binding in `incident-investigation.yaml` with a `when` guard that checks case context for `containmentApproved` OR `containmentAutonomous`. The approval handler / risk classifier sets the appropriate flag.
**Alternatives:**
- Two explicit paths — separate YAML bindings for gated and autonomous. More explicit but duplicates the execution binding.
- Event-driven (no YAML) — CDI events trigger execution. More flexible but the case plan doesn't show the containment execution step, making audit trail reconstruction harder.
**Rationale:** Single path keeps the case plan readable and avoids duplication. The `when` guard is the existing sequencing mechanism — all current bindings use it. The case context flags (`containmentApproved`, `containmentAutonomous`) provide clear branching semantics.
**Trade-offs:** The conditional guard checks two flags — slightly less obvious than two explicit paths. Mitigated by clear naming.
**Sources:** `incident-investigation.yaml` (existing `when` guard patterns), `SocCaseHub.augment()` (binding registration)
**Exploration:** quick
**Depends on:** D1 (separation creates the containment-approval step that sets `containmentApproved`)
**Status:** captured
