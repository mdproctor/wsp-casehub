# CrowdStrike Falcon Containment Connector — Design Spec

**Issue:** #52 — CrowdStrike Falcon containment connector — host isolation and endpoint wipe
**Parent:** Epic #50 — Connector-based containment runtimes (Batch 2)
**Date:** 2026-09-14

---

## Problem

`HttpContainmentExecutor` routes containment actions to HTTP connector endpoints, but only the `SimulatedContainmentConnector` exists. CrowdStrike Falcon is the first real EDR integration — it provides host isolation and endpoint wipe via REST API.

## Approach

In-process JAX-RS endpoint within SOC (D1), following the `SimulatedContainmentConnector` pattern. Receives `ContainmentRequest`, translates to CrowdStrike Falcon API calls, returns `ContainmentResponse`.

## CrowdStrike Falcon API

### Host Containment (isolate.host)

```
POST https://api.crowdstrike.com/devices/entities/devices-actions/v2?action_name=contain
Authorization: Bearer <oauth2-token>
Content-Type: application/json

{
  "ids": ["<device_id>"]
}
```

Response: `{"resources": [{"id": "...", ...}], "errors": []}`.

`ContainmentRequest.parameters` must include `deviceId` (the CrowdStrike device ID). The connector maps this to the `ids` array.

### Host Release (wipe.endpoint)

```
POST https://api.crowdstrike.com/devices/entities/devices-actions/v2?action_name=lift_containment
Authorization: Bearer <oauth2-token>
Content-Type: application/json

{
  "ids": ["<device_id>"]
}
```

Same API, different `action_name`. `wipe.endpoint` maps to `lift_containment` since CrowdStrike's wipe is a containment-lift (return to normal state). If a destructive wipe is needed later, a separate action type can be added.

### OAuth2 Authentication

CrowdStrike uses OAuth2 client credentials:

```
POST https://api.crowdstrike.com/oauth2/token
Content-Type: application/x-www-form-urlencoded

client_id=<id>&client_secret=<secret>
```

Returns `{"access_token": "...", "expires_in": 1799}` (30 min TTL).

## New Components

### CrowdStrikeContainmentConnector

JAX-RS endpoint at `/crowdstrike/containment` in `app/`.

```java
@Path("/crowdstrike/containment")
@ApplicationScoped
public class CrowdStrikeContainmentConnector {

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public ContainmentResponse execute(ContainmentRequest request) {
        // 1. Get/refresh OAuth2 token
        // 2. Map actionType → CrowdStrike action_name
        // 3. Extract deviceId from request.parameters
        // 4. POST to CrowdStrike API
        // 5. Map response → ContainmentResponse
    }
}
```

**Action mapping:**

| ContainmentRequest.actionType | CrowdStrike action_name |
|---|---|
| `isolate.host` | `contain` |
| `wipe.endpoint` | `lift_containment` |

**Required parameters:** `request.parameters().get("deviceId")` — the CrowdStrike device ID. If missing, return `ContainmentResponse(false, null, "Missing required parameter: deviceId", false, Map.of())`.

**Module:** `app/`
**Package:** `io.casehub.soc.connector.crowdstrike`

### CrowdStrikeOAuth2Client

Manages OAuth2 token lifecycle — request, cache, refresh.

```java
@ApplicationScoped
public class CrowdStrikeOAuth2Client {

    String getAccessToken() {
        // Return cached token if not expired (with 60s buffer)
        // Otherwise request new token via client credentials
    }
}
```

Token is cached in-memory with TTL from the `expires_in` response field minus a 60-second safety buffer. Thread-safe via `volatile` token reference — no lock contention on the hot path.

**Config:**
```properties
casehub.soc.crowdstrike.api-base=https://api.crowdstrike.com
casehub.soc.crowdstrike.client-id=${CROWDSTRIKE_CLIENT_ID}
casehub.soc.crowdstrike.client-secret=${CROWDSTRIKE_CLIENT_SECRET}
```

**Package:** `io.casehub.soc.connector.crowdstrike`

### Health Check

```java
@Path("/crowdstrike/health")
@ApplicationScoped
public class CrowdStrikeHealthCheck {

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response health() {
        // Attempt OAuth2 token request
        // 200 with {"status": "UP"} if successful
        // 503 with {"status": "DOWN", "reason": "..."} if not
    }
}
```

**Package:** `io.casehub.soc.connector.crowdstrike`

## Configuration

### application.properties additions

```properties
# CrowdStrike connector endpoint (for HttpContainmentExecutor routing)
casehub.soc.containment.endpoints.crowdstrike.url=http://localhost:${quarkus.http.port}/crowdstrike/containment
casehub.soc.containment.endpoints.crowdstrike.method=POST
casehub.soc.containment.endpoints.crowdstrike.timeout-seconds=30

# CrowdStrike API credentials (env vars in production)
casehub.soc.crowdstrike.api-base=https://api.crowdstrike.com
casehub.soc.crowdstrike.client-id=${CROWDSTRIKE_CLIENT_ID:}
casehub.soc.crowdstrike.client-secret=${CROWDSTRIKE_CLIENT_SECRET:}
```

In dev/test, the routing stays pointed at `sim` (existing config). In production, override `casehub.soc.containment.routing.isolate-host=crowdstrike` and `casehub.soc.containment.routing.wipe-endpoint=crowdstrike`.

## Testing Strategy

### Unit tests

- **CrowdStrikeContainmentConnectorTest** — action mapping, parameter validation (missing deviceId), response mapping for success/error/unexpected formats
- **CrowdStrikeOAuth2ClientTest** — token caching, expiry refresh, error handling (401, network failure)

### Contract tests (WireMock)

- **CrowdStrikeContainmentWireMockTest** (`@QuarkusTest` + WireMock) — full round-trip:
  - Stub OAuth2 token endpoint → return valid token
  - Stub device actions endpoint → return success/error
  - POST `ContainmentRequest` to `/crowdstrike/containment`
  - Verify correct CrowdStrike API calls were made (auth header, request body)
  - Verify `ContainmentResponse` mapping

**WireMock scenarios:**
1. Successful host isolation — OAuth2 token + contain action → success
2. Successful endpoint wipe — OAuth2 token + lift_containment → success
3. CrowdStrike returns error — device not found → failure (retryable=false)
4. CrowdStrike returns 429 — rate limited → failure (retryable=true)
5. CrowdStrike returns 500 — transient error → failure (retryable=true)
6. OAuth2 token expired mid-request — 401 on action, refresh, retry → success
7. OAuth2 token request fails — 401 on token endpoint → failure (retryable=true)
8. Missing deviceId parameter → failure (retryable=false)

### Integration test

Re-enable `ConnectorContainmentIntegrationTest` (currently `@Disabled` due to #34 which is now fixed). This test validates the full pipeline through the simulated connector — it doesn't need CrowdStrike-specific tests since those are covered by WireMock.

## Failure Modes

| Failure | Connector behavior |
|---|---|
| Missing `deviceId` parameter | `ContainmentResponse(false, null, "Missing required parameter: deviceId", false, {})` |
| OAuth2 token request fails | `ContainmentResponse(false, null, "CrowdStrike authentication failed: <reason>", true, {})` |
| CrowdStrike API 200 with errors | `ContainmentResponse(false, null, <first error message>, false, {"crowdstrike_errors": [...]})` |
| CrowdStrike API 429 | `ContainmentResponse(false, null, "CrowdStrike rate limited", true, {})` |
| CrowdStrike API 4xx | `ContainmentResponse(false, null, "CrowdStrike error: <status>", false, {})` |
| CrowdStrike API 5xx | `ContainmentResponse(false, null, "CrowdStrike server error: <status>", true, {})` |
| Network timeout/connection refused | `ContainmentResponse(false, null, "CrowdStrike unreachable: <reason>", true, {})` |
| Success | `ContainmentResponse(true, "Host <deviceId> contained", null, false, {"crowdstrike_device_id": "...", "action_name": "..."})` |

## References

- `SimulatedContainmentConnector.java:19` — reference implementation for connector pattern
- `HttpContainmentExecutor.java:25` — executor that routes to this connector
- `ContainmentRequest.java:5` / `ContainmentResponse.java:6` — API contract
- `ContainmentEndpointResolver.java` — config-driven endpoint resolution
- `docs/specs/issue-50-connector-containment-runtimes/2026-09-14-connector-containment-runtimes-design.md` — parent spec
- CrowdStrike Falcon API: `POST /devices/entities/devices-actions/v2` — host containment/release
- CrowdStrike OAuth2: `POST /oauth2/token` — client credentials flow
