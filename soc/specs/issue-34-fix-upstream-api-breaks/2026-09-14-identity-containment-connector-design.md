# Identity Containment Connector (Okta + Microsoft Graph) — Design Spec

**Issue:** #54 — Okta/Azure AD containment connector — credential revocation and account disabling
**Parent:** Epic #50 — Connector-based containment runtimes (Batch 4)
**Date:** 2026-09-14

---

## Problem

Three containment action types — `disable.user.account`, `revoke.credentials`, `rotate.api.key` — route to the simulated connector. Real containment for identity-based threats requires calling an identity provider: Okta or Microsoft Entra ID (Azure AD).

## Approach

In-process JAX-RS endpoint at `/identity/containment` (D1), following the CrowdStrike/Palo Alto connector pattern. Abstracts both Okta and Microsoft Graph behind a strategy pattern (D6), with config-driven provider selection (D7).

A single config property (`casehub.soc.identity.provider=okta` or `graph`) selects the active identity provider at startup. Internally, each provider implements the same `IdentityProvider` interface with provider-specific HTTP calls and auth.

## Identity Provider APIs

### Okta Users API

**Auth:** SSWS API token in the `Authorization` header:

```
Authorization: SSWS <api-token>
```

Long-lived token, no refresh lifecycle.

**disable.user.account:**

```
POST https://<domain>/api/v1/users/<userId>/lifecycle/suspend
Authorization: SSWS <token>
```

Response 200: `{}` (empty body on success).
Response 404: user not found.

**revoke.credentials:**

```
DELETE https://<domain>/api/v1/users/<userId>/sessions
Authorization: SSWS <token>
```

Response 204: sessions cleared.
Response 404: user not found.

**rotate.api.key:**

```
POST https://<domain>/api/v1/apps/<appId>/credentials/keys/generate?validityYears=1
Authorization: SSWS <token>
```

Response 201: `{"kid": "...", "created": "...", ...}` — new key metadata.
Response 404: app not found.

### Microsoft Graph API

**Auth:** OAuth2 client credentials flow (same pattern as CrowdStrike):

```
POST https://login.microsoftonline.com/<tenantId>/oauth2/v2.0/token
Content-Type: application/x-www-form-urlencoded

client_id=<id>&client_secret=<secret>&scope=https://graph.microsoft.com/.default&grant_type=client_credentials
```

Returns `{"access_token": "...", "expires_in": 3599}`.

**disable.user.account:**

```
PATCH https://graph.microsoft.com/v1.0/users/<userId>
Authorization: Bearer <token>
Content-Type: application/json

{"accountEnabled": false}
```

Response 204: account disabled.
Response 404: user not found.

**revoke.credentials:**

```
POST https://graph.microsoft.com/v1.0/users/<userId>/revokeSignInSessions
Authorization: Bearer <token>
```

Response 200: `{"value": true}`.
Response 404: user not found.

**rotate.api.key:**

```
POST https://graph.microsoft.com/v1.0/applications/<appId>/addPassword
Authorization: Bearer <token>
Content-Type: application/json

{"passwordCredential": {"displayName": "casehub-rotated"}}
```

Response 200: `{"secretText": "...", "keyId": "...", ...}` — new credential.
Response 404: application not found.

## New Components

### IdentityProvider (interface)

Strategy interface for identity provider operations.

```java
public interface IdentityProvider {
    IdentityResult disableAccount(String userId);
    IdentityResult revokeSessions(String userId);
    IdentityResult rotateApiKey(String appId);
    IdentityResult healthCheck();
}
```

`IdentityResult` is a simple record:
```java
public record IdentityResult(boolean success, String details, String errorReason,
                               boolean retryable, Map<String, Object> metadata) {}
```

**Package:** `io.casehub.soc.connector.identity`

### OktaIdentityProvider

Implements `IdentityProvider` via Okta Users API. Uses SSWS API token auth (stateless — no token refresh).

```java
public class OktaIdentityProvider implements IdentityProvider {
    // POST /api/v1/users/{userId}/lifecycle/suspend
    // DELETE /api/v1/users/{userId}/sessions
    // POST /api/v1/apps/{appId}/credentials/keys/generate
}
```

**Package:** `io.casehub.soc.connector.identity`

### GraphIdentityProvider

Implements `IdentityProvider` via Microsoft Graph API. Uses OAuth2 client credentials flow with in-memory token caching (same pattern as `CrowdStrikeOAuth2Client`).

```java
public class GraphIdentityProvider implements IdentityProvider {
    // PATCH /v1.0/users/{userId} {"accountEnabled": false}
    // POST /v1.0/users/{userId}/revokeSignInSessions
    // POST /v1.0/applications/{appId}/addPassword
}
```

Token caching: OAuth2 token cached with TTL from `expires_in` minus 60-second buffer. Same `volatile` approach as CrowdStrike.

**Package:** `io.casehub.soc.connector.identity`

### IdentityContainmentConnector

JAX-RS endpoint at `/identity/containment` in `app/`.

```java
@Path("/identity/containment")
@ApplicationScoped
public class IdentityContainmentConnector {

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public ContainmentResponse execute(ContainmentRequest request) {
        // 1. Validate action type and required parameters
        // 2. Dispatch to IdentityProvider method
        // 3. Map IdentityResult → ContainmentResponse
    }
}
```

**Action mapping:**

| ContainmentRequest.actionType | IdentityProvider method | Required parameters |
|---|---|---|
| `disable.user.account` | `disableAccount(userId)` | `userId` |
| `revoke.credentials` | `revokeSessions(userId)` | `userId` |
| `rotate.api.key` | `rotateApiKey(appId)` | `appId` |

**Module:** `app/`
**Package:** `io.casehub.soc.connector.identity`

### IdentityHealthCheck

```java
@Path("/identity/health")
@ApplicationScoped
public class IdentityHealthCheck {

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response health() {
        // Call provider.healthCheck()
        // 200 {"status": "UP", "provider": "okta|graph"} if ok
        // 503 {"status": "DOWN", "reason": "..."} if not
    }
}
```

**Package:** `io.casehub.soc.connector.identity`

### IdentityConnectorConfig

CDI producer. Reads config, creates the correct provider.

```java
@ApplicationScoped
public class IdentityConnectorConfig {

    @Produces
    @Singleton
    public IdentityProvider identityProvider(
            @ConfigProperty(name = "casehub.soc.identity.provider",
                             defaultValue = "okta") String provider,
            ...) {
        return switch (provider) {
            case "okta" -> new OktaIdentityProvider(apiBase, apiToken);
            case "graph" -> new GraphIdentityProvider(apiBase, tenantId, clientId, clientSecret);
            default -> IdentityProvider.unconfigured();
        };
    }
}
```

**Package:** `io.casehub.soc.connector.identity`

## Configuration

### application.properties additions

```properties
# Identity containment connector endpoint (for HttpContainmentExecutor routing)
casehub.soc.containment.endpoints.okta.url=http://localhost:${quarkus.http.port}/identity/containment
casehub.soc.containment.endpoints.okta.method=POST
casehub.soc.containment.endpoints.okta.timeout-seconds=30

# Identity provider config
casehub.soc.identity.provider=okta
casehub.soc.identity.api-base=https://dev-123456.okta.com
casehub.soc.identity.api-token=${OKTA_API_TOKEN:}

# For Microsoft Graph (uncomment and set provider=graph):
# casehub.soc.identity.api-base=https://graph.microsoft.com
# casehub.soc.identity.tenant-id=${AZURE_TENANT_ID:}
# casehub.soc.identity.client-id=${AZURE_CLIENT_ID:}
# casehub.soc.identity.client-secret=${AZURE_CLIENT_SECRET:}
```

In dev/test, routing stays pointed at `sim`. In production, override `casehub.soc.containment.routing.disable-user-account=okta`, `casehub.soc.containment.routing.revoke-credentials=okta`, `casehub.soc.containment.routing.rotate-api-key=okta`.

## Testing Strategy

### Unit tests

- **IdentityContainmentConnectorTest** — action mapping, parameter validation (missing userId, missing appId, unsupported action), response mapping
- **OktaIdentityProviderTest** — unconfigured state, URL construction
- **GraphIdentityProviderTest** — unconfigured state, URL construction, token caching

### Contract tests (WireMock)

- **IdentityContainmentWireMockTest** — two test inner groups: Okta scenarios and Graph scenarios

**Okta WireMock scenarios:**
1. Successful account disable — `POST /lifecycle/suspend` → 200
2. Successful session revocation — `DELETE /sessions` → 204
3. Successful API key rotation — `POST /credentials/keys/generate` → 201
4. User not found → 404 → failure (retryable=false)
5. Server error → 500 → failure (retryable=true)
6. Rate limited → 429 → failure (retryable=true)

**Graph WireMock scenarios:**
1. Successful account disable — OAuth2 token + `PATCH /users/{id}` → 204
2. Successful session revocation — `POST /revokeSignInSessions` → 200
3. Successful API key rotation — `POST /addPassword` → 200
4. User not found → 404 → failure (retryable=false)
5. OAuth2 token request fails → failure (retryable=true)
6. Server error → 500 → failure (retryable=true)

## Failure Modes

| Failure | Connector behavior |
|---|---|
| Missing required parameter (`userId`/`appId`) | `ContainmentResponse(false, null, "Missing required parameter: <param>", false, {})` |
| Provider not configured | `ContainmentResponse(false, null, "Identity provider not configured", false, {})` |
| User/app not found (404) | `ContainmentResponse(false, null, "<provider>: user/app not found", false, {"provider": "okta\|graph"})` |
| Auth failure (Okta 401, Graph OAuth2 fail) | `ContainmentResponse(false, null, "<provider> authentication failed", false, {})` |
| Rate limited (429) | `ContainmentResponse(false, null, "<provider> rate limited", true, {})` |
| Server error (5xx) | `ContainmentResponse(false, null, "<provider> server error: <status>", true, {})` |
| Unreachable | `ContainmentResponse(false, null, "<provider> unreachable: <reason>", true, {})` |
| Success (disable) | `ContainmentResponse(true, "User <userId> disabled via <provider>", null, false, {"provider": "...", "identity_user_id": "..."})` |
| Success (revoke) | `ContainmentResponse(true, "Sessions revoked for <userId> via <provider>", null, false, {"provider": "..."})` |
| Success (rotate) | `ContainmentResponse(true, "API key rotated for <appId> via <provider>", null, false, {"provider": "...", "identity_key_id": "..."})` |

## References

- `CrowdStrikeContainmentConnector.java:28` — reference connector (same JAX-RS pattern)
- `CrowdStrikeOAuth2Client.java:13` — reference OAuth2 token caching (reused by GraphIdentityProvider)
- `PaloAltoContainmentConnector.java` — reference connector (action mapping pattern)
- `PaloAltoConfig.java` — reference CDI producer with Optional config
- `HttpContainmentExecutor.java:25` — executor that routes to this connector
- `ContainmentRequest.java:5` / `ContainmentResponse.java:6` — API contract
- `specs/issue-50-connector-containment-runtimes/2026-09-14-connector-containment-runtimes-design.md` — parent spec (Batch 4)
- Okta Users API: `POST /api/v1/users/{userId}/lifecycle/suspend`
- Okta Sessions API: `DELETE /api/v1/users/{userId}/sessions`
- Okta Apps API: `POST /api/v1/apps/{appId}/credentials/keys/generate`
- Microsoft Graph: `PATCH /v1.0/users/{userId}`, `POST /revokeSignInSessions`, `POST /addPassword`
- Microsoft Identity Platform: OAuth2 client credentials flow
