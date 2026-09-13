# Demo Infrastructure: Alert Injection + Scenario Scripts — Design Spec

**Date:** 2026-08-29
**Issue:** casehubio/soc#33
**Branch:** issue-33-demo-infrastructure

---

## Overview

Adds a REST endpoint to inject simulated SIEM alerts into the RAS pipeline, plus scenario YAML files for the pages script library. The endpoint wraps the proven pattern from `AlertToCaseIntegrationTest` — constructing a CloudEvent and firing it through `SituationEvaluator.evaluate()`. No new domain classes, no new services — thin wiring of existing RAS APIs.

**Scope:** One REST resource, one integration test, 2-3 scenario YAML files.

---

## Alert Injection Endpoint

### `POST /api/soc/demo/inject-alert`

`@RolesAllowed("soc-demo-admin")`

**Request:**

```json
{
  "eventType": "soc.alert.siem.crowdstrike",
  "severity": "CRITICAL",
  "source": "10.0.1.42",
  "rule": "credential-harvesting",
  "correlationKey": "host-10.0.1.42"
}
```

| Field | Required | Default | Description |
|---|---|---|---|
| `eventType` | yes | — | Must match a registered RAS situation event type |
| `severity` | yes | — | CRITICAL, HIGH, MEDIUM, LOW, INFORMATIONAL |
| `source` | no | `demo-source` | Alert source identifier (host, IP, sensor) |
| `rule` | no | `demo-rule` | Alert rule name |
| `correlationKey` | no | random UUID | RAS correlation key for event grouping |

**Response:**

```json
{
  "situationId": "soc-siem-alert-critical",
  "eventId": "uuid",
  "correlationKey": "host-10.0.1.42",
  "evaluated": true
}
```

If `eventType` doesn't match any registered situation: 400 with `"No situation registered for event type: <type>"`.

### Implementation

```java
@Path("/api/soc/demo")
@ApplicationScoped
@RolesAllowed("soc-demo-admin")
public class SocDemoResource {

    @Inject SituationEvaluator evaluator;
    @Inject SituationDefinitionRegistry registry;
    @Inject CurrentPrincipal currentPrincipal;

    @POST @Path("/inject-alert")
    public Response injectAlert(Map<String, String> body) {
        String eventType = body.get("eventType");
        String severity = body.getOrDefault("severity", "HIGH");
        String source = body.getOrDefault("source", "demo-source");
        String rule = body.getOrDefault("rule", "demo-rule");
        String correlationKey = body.getOrDefault("correlationKey",
            UUID.randomUUID().toString());

        List<SituationRegistration> registrations =
            registry.findByEventType(eventType);
        if (registrations.isEmpty()) {
            return Response.status(400)
                .entity(Map.of("error",
                    "No situation registered for event type: " + eventType))
                .build();
        }

        String eventId = UUID.randomUUID().toString();
        CloudEvent event = CloudEventBuilder.v1()
            .withId(eventId)
            .withSource(URI.create("soc://demo"))
            .withType(eventType)
            .withExtension("alertseverity", severity)
            .withExtension("alertsource", source)
            .withExtension("alertrule", rule)
            .withExtension("tenancyid",
                currentPrincipal.tenancyId())
            .build();

        SituationRegistration reg = registrations.getFirst();
        evaluator.evaluate(event, reg.definition(),
            correlationKey, currentPrincipal.tenancyId());

        return Response.ok(Map.of(
            "situationId", reg.definition().situationId(),
            "eventId", eventId,
            "correlationKey", correlationKey,
            "evaluated", true
        )).build();
    }
}
```

### Validation

- `eventType` required — 400 if missing
- `severity` validated against `AlertSeverity` enum — 400 on invalid value
- All other fields have safe defaults

---

## Scenario YAML Files

Scenario scripts for the pages script library. Located at `app/src/main/resources/scenarios/`.

### Scenario 1: Critical SIEM Alert Investigation

```yaml
scenario: soc-critical-alert-investigation
description: "Inject a critical CrowdStrike alert and watch the investigation pipeline"
steps:
  - name: inject-alert
    delivery: rest
    method: POST
    url: /api/soc/demo/inject-alert
    body:
      eventType: soc.alert.siem.crowdstrike
      severity: CRITICAL
      source: "10.0.1.42"
      rule: credential-harvesting
      correlationKey: "demo-host-1"

  - navigate: /
  - click:
      role: tab
      name: "Incidents"
  - wait:
      role: row
      name: "CRITICAL"
      timeout: 10000

  - click:
      role: tab
      name: "Workbench"
  - wait:
      role: listitem
      timeout: 10000

  - click:
      role: tab
      name: "Trust"

  - click:
      role: tab
      name: "Compliance"
```

### Scenario 2: Brute Force Detection

```yaml
scenario: soc-brute-force-detection
description: "Inject 5+ failed login events to trigger brute force correlation"
steps:
  - name: login-1
    delivery: rest
    method: POST
    url: /api/soc/demo/inject-alert
    body:
      eventType: soc.alert.auth.failed-login
      severity: MEDIUM
      source: "192.168.1.100"
      rule: failed-login-attempt
      correlationKey: "192.168.1.100"

  - name: login-2
    delivery: rest
    method: POST
    url: /api/soc/demo/inject-alert
    body:
      eventType: soc.alert.auth.failed-login
      severity: MEDIUM
      source: "192.168.1.100"
      rule: failed-login-attempt
      correlationKey: "192.168.1.100"

  - name: login-3
    delivery: rest
    method: POST
    url: /api/soc/demo/inject-alert
    body:
      eventType: soc.alert.auth.failed-login
      severity: MEDIUM
      source: "192.168.1.100"
      rule: failed-login-attempt
      correlationKey: "192.168.1.100"

  - name: login-4
    delivery: rest
    method: POST
    url: /api/soc/demo/inject-alert
    body:
      eventType: soc.alert.auth.failed-login
      severity: MEDIUM
      source: "192.168.1.100"
      rule: failed-login-attempt
      correlationKey: "192.168.1.100"

  - name: login-5
    delivery: rest
    method: POST
    url: /api/soc/demo/inject-alert
    body:
      eventType: soc.alert.auth.failed-login
      severity: MEDIUM
      source: "192.168.1.100"
      rule: failed-login-attempt
      correlationKey: "192.168.1.100"

  - navigate: /
  - click:
      role: tab
      name: "Incidents"
  - wait:
      role: row
      timeout: 15000
```

**Note:** The scenario executor's `delivery: rest` type may need verification against the pages scenario spec. If only `graphql` is supported for backend steps, wrap the inject-alert call as a GraphQL mutation or use a thin GraphQL adapter. The endpoint itself stays REST — only the scenario delivery mechanism changes.

---

## Testing

| Level | What | How |
|---|---|---|
| Integration | Inject alert, verify case created | `@QuarkusTest` — same pattern as `AlertToCaseIntegrationTest` but via REST endpoint |
| Integration | Invalid event type returns 400 | `@QuarkusTest` with RestAssured |
| Integration | Missing severity returns 400 | `@QuarkusTest` with RestAssured |

---

## Files Changed

| File | Change |
|---|---|
| `app/src/main/java/io/casehub/soc/rest/SocDemoResource.java` | New — alert injection endpoint |
| `app/src/test/java/io/casehub/soc/rest/SocDemoResourceTest.java` | New — integration tests |
| `app/src/main/resources/scenarios/soc-critical-alert-investigation.yaml` | New — demo scenario |
| `app/src/main/resources/scenarios/soc-brute-force-detection.yaml` | New — demo scenario |

---

## References

- `app/src/test/java/io/casehub/soc/integration/AlertToCaseIntegrationTest.java` — proven injection pattern
- `api/src/main/java/io/casehub/soc/detection/SiemAlertGanglion.java` — CloudEvent extensions (alertseverity, alertsource, alertrule)
- `app/src/main/resources/META-INF/ras-situations.yaml` — situation definitions (soc-siem-alert-critical, soc-brute-force-by-source)
- `app/src/main/resources/soc/incident-investigation.yaml` — case definition triggered by RAS
- `io.casehub.ras.runtime.SituationEvaluator` — evaluate(CloudEvent, SituationDefinition, correlationKey, tenancyId)
- `io.casehub.ras.runtime.SituationDefinitionRegistry` — findByEventType()
- Pages scenario executor (epic #408) — YAML format and execution engine
