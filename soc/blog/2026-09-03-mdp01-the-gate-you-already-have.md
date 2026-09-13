---
title: "The Gate You Already Have"
date: 2026-09-03
author: mdp
entry_type: note
subtype: diary
project: casehub-soc
tags: [containment, compliance, spi, design-review]
series: casehub-soc
---

# The Gate You Already Have

The containment pipeline was supposed to be the most complex piece of the SOC so far — gating irreversible actions through human approval, writing tamper-evident audit entries, plugging in actual EDR and firewall integrations later. The plan had three new components just for the gate mechanism alone: a worker to observe the risk classification, a humanTask binding for the approval WorkItem, and a handler to feed the decision back into the case.

Claude caught it in the design review: every one of those components duplicates what the engine already does.

When a worker emits a `PlannedAction`, the engine runs it through `ChainedActionRiskClassifier`. If the result is `GateRequired`, the engine pauses the case, creates a WorkItem via `JudgmentScheduler`, and routes it to the right approver group. When the WorkItem resolves — approved, rejected, expired — `ActionGateApprovedHandler` or `ActionGateRejectedHandler` writes the decision to case context and re-fires `CONTEXT_CHANGED`. The entire gate lifecycle is built-in. We were proposing to rebuild it from scratch, with our own CDI observers and YAML bindings, running in parallel with the engine's own mechanism. Two WorkItems for the same approval. Two context signals for the same decision.

The revised design deleted three components and kept the one that matters: `RuleContainmentExecutionWorker`, which reads the engine's `actionGateApproved` signal and delegates to a pluggable `ContainmentExecutor` SPI. The default implementation logs the action. Real integrations — CrowdStrike for host isolation, Palo Alto for network segmentation — swap in via CDI `@Alternative`. The SPI includes a timeout contract and a `retryable` flag on failure results, because real containment targets are external systems that hang and flake.

The other piece that survived cleanly: the compliance audit trail. Three ledger entries per containment action — gate decision, approval outcome, execution result — each linked by `causedByEntryId`. DORA wants detection-to-containment elapsed time. SOC2 wants who approved what. GDPR Art. 22 wants the full decision chain for automated actions affecting individuals. Three entries, three compliance requirements, one `causedByEntryId` chain.

The ordering matters and we got it wrong in the first draft. The original spec had analyst triage *after* containment execution — which means a false positive alert would get its host isolated before anyone confirmed it was a real incident. The review caught this too. The final pipeline runs triage first: the analyst confirms the threat, *then* the approved containment executes. For the gated path, the SOC manager already approved the specific action via the engine's WorkItem — but the Tier-1 analyst still confirms the incident is real before anything irreversible happens.

The `confidenceScore` fix was a small thing with outsized impact. `SocActionType.ROTATE_API_KEY` uses `GatePolicy.CONFIDENCE_THRESHOLD`, but `RuleContainmentRecommendationWorker` wasn't including `confidenceScore` in the `PlannedAction` parameters. The classifier fell through to `missingContext()` — forced `GateRequired` every time. One missing map key turned every API key rotation into a human approval gate.

Fifteen new Java files, 315 tests passing, one upstream dependency issue filed. The containment pipeline is the first piece of the SOC that actually does something consequential — and the design review that simplified it was worth more than the implementation that followed.
