---
layout: post
title: "Making it real"
date: 2026-08-29
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [soc, demo, ras, scenario-executor, deployment, api-drift, flyway]
series: issue-33-demo-infrastructure
---

# Making it real

Four sidebar tabs built. 283 tests green. Every compliance badge, every Merkle proof, every GDPR erasure form wired to its endpoint. And then the obvious question: can someone actually run this?

The answer was no. Not because anything was missing in the application logic — the workers are real, the case definition fires through cbr-retrieval into ioc-enrichment into attck-mapping into containment-recommendation into analyst-review. The RAS situations are defined. The ganglion classifies SIEM alerts by severity. But nothing triggers the first event. The system is a loaded gun with no trigger.

The pages team landed the scenario executor while we were building the compliance tab — a YAML-scripted automation engine that drives browser interactions and backend mutations. It's designed for exactly this: inject an alert, wait for the engine to create a case, navigate through the UI verifying data appears. The SOC app just needs to meet it halfway with an injection endpoint.

`POST /api/soc/demo/inject-alert` constructs a CloudEvent with the right extensions (`alertseverity`, `alertsource`, `alertrule`) and fires it through `SituationEvaluator.evaluate()`. The exact pattern from `AlertToCaseIntegrationTest` — proven code, thin REST wrapper. Role-gated with `soc-demo-admin` so it's available in staging and training, not just dev mode.

Two scenario YAML scripts: a critical CrowdStrike alert investigation that walks through all four tabs, and a brute force detection that correlates five failed logins from the same source. These go into the pages script library — selectable automations, not hidden dev tools.

Then reality hit harder than expected. A week of dependency drift while focused on the web UI left six upstream API breaks accumulating silently. `CbrCase.withOutcome` changed its signature from `Double` to `Confidence`. `Worker.capabilityNames()` became `capabilities()`. `PlanItemCallerRef` was refactored into `PlanItemRef`. And `HumanTaskTarget` was removed from `BindingTarget`'s sealed hierarchy entirely — replaced by `JudgmentTarget` with a `HumanRoutingConfig` for candidate routing. The YAML `humanTask:` key still works, but the parser now produces a `JudgmentTarget` internally. Seven integration tests had been disabled waiting for the fix.

The nastiest one was a Flyway migration. The work module recently consolidated its V1 through V44 migrations into a single V1 initial schema, but left a stale V42 delta referencing the old table name `work_items` instead of `work_item`. Production databases — migrated incrementally — had both names. Fresh H2 test databases only had `work_item` from the consolidated V1. Every `@QuarkusTest` in the SOC app failed at startup with "Table WORK_ITEMS not found." The mismatch was invisible in the work module's own tests because they'd been migrated incrementally too.

All six API breaks fixed, the Flyway migration corrected in the local SNAPSHOT, and 287 tests pass — zero skipped. The seven analyst-review integration tests are back, now asserting against `JudgmentTarget` and `HumanRoutingConfig` instead of the deprecated `HumanTaskTarget`. The demo endpoint returns proper JSON error bodies for validation failures, which the original `BadRequestException` approach silently didn't.

The Flyway gotcha is worth calling out: after any migration consolidation, grep the remaining deltas for the old table names. The failure only surfaces in downstream apps running `clean-at-start=true` — which could be a different repo, a different team, weeks later. Filed upstream for the fix.
