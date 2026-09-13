# CBR Retrieval: Closing the Feedback Loop — Design Spec

**Date:** 2026-09-01
**Issue:** casehubio/soc#35
**Branch:** issue-35-cbr-retrieval
**Covers:** #36 (worker), #37 (seed data), #38 (e2e test), #39 (tuning)

---

## Overview

Registers a CBR retrieval worker that satisfies the `cbr-retrieval` capability binding, closing the loop between past incident resolutions and future triage. The infrastructure is fully built — `SocCbrRetrieveService`, `SocCbrRetainService`, `SocIncidentCbrCase`, case type registration, schema registration, REST endpoint. The only gap is a registered worker that the engine can dispatch.

**Scope:** One worker, one seed data loader, one integration test, one tuning pass. No new domain classes or SPIs — pure wiring of existing APIs.

---

## #36 — CBR Retrieval Worker

### Architecture

`RuleCbrRetrievalWorker` follows the same pattern as `RuleIocEnrichmentWorker` — a final class with a static `create()` factory method returning a `Worker`. The difference: it takes `SocCbrRetrieveService` as a parameter (D1).

### Wiring chain

```
SocCaseHub (@ApplicationScoped)
  └─ @Inject SocCbrRetrieveService
  └─ new SocInvestigationCaseDescriptor(llmModel, retrieveService)
       └─ RuleCbrRetrievalWorker.create(retrieveService)
```

`SocCaseHub.augment()` already injects `ChatModel` optionally (for LLM workers). Adding `SocCbrRetrieveService` as a second CDI injection follows the same pattern.

### Worker function

```java
public static Worker create(SocCbrRetrieveService retrieveService) {
    return Worker.builder()
        .name("rule-cbr-retrieval")
        .capabilityName("cbr-retrieval")
        .function((Map<String, Object> input) -> {
            List<Map<String, Object>> results =
                retrieveService.retrieve(input, DEFAULT_TENANT);
            String summary = results.isEmpty()
                ? "No similar past incidents found"
                : results.size() + " similar incident(s) retrieved";
            return WorkerResult.of(Map.of(
                "retrievedIncidents", results,
                "summary", summary));
        })
        .build();
}
```

The input map is the capability's `inputProjection: "{ alert: .alert }"` — the worker receives `{ "alert": { ... } }` and passes it to `SocCbrRetrieveService.retrieve()`.

### TenantId handling

`SocCbrRetrieveService.retrieve()` requires a `tenantId`. For pre-release, use the default tenant constant (`278776f9-e1b0-46fb-9032-8bddebdcf9ce` — the same one in Flyway migrations and test fixtures). When multi-tenancy is production-ready, this will be sourced from `WorkerScope` or case context — but the engine doesn't yet propagate tenant context to workers. A constant is the right design for now; a TODO marks the future change point.

### Registration order

The cbr-retrieval binding fires first (`.alert != null and .retrievedIncidents == null`). The worker should be registered before the IOC enrichment workers so bootstrap routing selects it first. Add at position 0 in `SocInvestigationCaseDescriptor.workers()`.

---

## #37 — Seed Data

### `SocCbrSeedDataLoader`

`@ApplicationScoped` with `void onStartup(@Observes StartupEvent event)`. Gated by `@IfBuildProfile("dev")` OR `@IfBuildProfile("test")` — does not load in production.

Populates `CbrCaseMemoryStore` with 5 representative historical incidents:

| Alert type | Source | Outcome | Playbook |
|---|---|---|---|
| credential-harvesting | crowdstrike | CONFIRM_SEVERITY | isolate-host |
| brute-force | auth-service | DOWNGRADE | block-ip |
| malware-execution | crowdstrike | ESCALATE | escalate-tier2 |
| phishing | email-gateway | FALSE_POSITIVE | — |
| lateral-movement | network-ids | CONFIRM_SEVERITY | segment-network |

Each incident is a `SocIncidentCbrCase` with realistic features: alert type, source system, ATT&CK technique IDs, IOC types, severity outcome, containment outcome, playbook, and a plausible investigation duration.

Features use `FeatureValue.string()` and `FeatureValue.stringList()` matching `SocIncidentCbrCase.buildFeatureMap()` conventions.

---

## #38 — E2E Integration Test

### `CbrRetrievalIntegrationTest`

`@QuarkusTest` that verifies the full CBR lifecycle:

1. **Seed** — call `SocCbrRetainService` directly to store one historical incident (credential-harvesting from crowdstrike, resolved with CONFIRM_SEVERITY)
2. **Inject** — POST to `/api/soc/demo/inject-alert` with a similar credential-harvesting alert
3. **Verify retrieval** — assert the case context's `retrievedIncidents` is non-empty (the CBR worker ran and returned the seeded incident)
4. **Verify downstream** — assert `iocEnrichment` and `attckMapping` are populated (downstream workers received the retrieved context)
5. **Verify retain** — after the case completes, assert a second incident is stored in the CBR memory

Step 3 is the critical assertion — if `retrievedIncidents` is populated, the CBR worker is registered, dispatched, and returned results.

### Challenges

- **Async engine**: case processing is async (virtual threads). The test needs to poll or wait for the case context to reach the expected state.
- **No clearAll on InMemoryCbrCaseMemoryStore** (GE-20260716-986cd1): test isolation requires a new store instance or careful ordering. Using `@QuarkusTest` with `clean-at-start=true` handles this via fresh container per test class.
- **Case completion**: the test needs the analyst-review binding to complete. Either mock it or use the `JudgmentTarget` auto-resolve mechanism if available.

---

## #39 — Similarity Tuning

### Current defaults

| Parameter | Value | Rationale |
|---|---|---|
| `TOP_K` | 5 | Return up to 5 similar incidents |
| `MIN_SIMILARITY` | 0.3 | Low threshold — show marginally similar incidents for triage context |

### Tuning approach

Parameterised test with the 5 seed incidents as the corpus. For each alert type, inject a similar alert and verify:
- **Precision**: the most similar seed incident ranks first
- **Recall**: alerts of the same type retrieve their matching seed incident above MIN_SIMILARITY
- **Separation**: dissimilar alert types score below MIN_SIMILARITY

If the defaults don't produce clean separation, adjust `MIN_SIMILARITY`. The feature extraction (`extractRetrievalFeatures`) uses `alertType`, `sourceSystem`, `severity`, and `alertDescription` — verify these are sufficient discriminators.

---

## Files Changed

| File | Change |
|---|---|
| `app/src/main/java/io/casehub/soc/worker/RuleCbrRetrievalWorker.java` | New — CBR retrieval worker |
| `app/src/main/java/io/casehub/soc/engine/SocInvestigationCaseDescriptor.java` | Add SocCbrRetrieveService parameter, register CBR worker |
| `app/src/main/java/io/casehub/soc/engine/SocCaseHub.java` | Inject SocCbrRetrieveService, pass to descriptor |
| `app/src/main/java/io/casehub/soc/engine/cbr/SocCbrSeedDataLoader.java` | New — seed data for dev/test |
| `app/src/test/java/io/casehub/soc/worker/RuleCbrRetrievalWorkerTest.java` | New — unit tests |
| `app/src/test/java/io/casehub/soc/integration/CbrRetrievalIntegrationTest.java` | New — e2e lifecycle test |
| `app/src/test/java/io/casehub/soc/engine/SocInvestigationCaseDescriptorTest.java` | Add cbr-retrieval capability assertion |

---

## References

- `SocCbrRetrieveService.java` — existing retrieve service (no changes)
- `SocCbrRetainService.java` — existing retain service (no changes)
- `SocIncidentCbrCase.java` — CBR case record with feature extraction
- `RuleIocEnrichmentWorker.java` — worker pattern to follow
- `incident-investigation.yaml` — cbr-retrieval capability and binding definition
- GE-20260716-986cd1 — InMemoryCbrCaseMemoryStore test isolation gotcha
- GE-20260804-0e1509 — FeatureValue type naming gotcha
- GE-20260804-7bd9f4 — ScoredCbrCase constructor parameter order
- GE-20260612-bd3b4d — Degenerate CBR technique (trust-scored routing)
