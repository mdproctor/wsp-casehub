# Identity Containment Connector Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #54 — Okta/Azure AD containment connector — credential revocation and account disabling
**Issue group:** #34, #52, #53, #54

**Goal:** Build a dual-provider identity containment connector that receives `ContainmentRequest` via JAX-RS, dispatches to Okta Users API or Microsoft Graph API based on config, and returns `ContainmentResponse`.

**Architecture:** JAX-RS endpoint at `/identity/containment` following the CrowdStrike/Palo Alto connector pattern. Strategy pattern internally: `IdentityProvider` interface with `OktaIdentityProvider` (SSWS token auth) and `GraphIdentityProvider` (OAuth2 client credentials with token caching). Config-driven provider selection at startup. WireMock stubs for contract tests against both APIs.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x WebClient (already available), WireMock (already in test scope), AssertJ

## Global Constraints

- Java 21 source on Java 26 JVM
- Quarkus 3.32.2
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
- Package: `io.casehub.soc.connector.identity`
- All commits reference `Refs #54`
- No new runtime dependencies — Vert.x WebClient is already transitive
- WireMock already in test scope (added in #52)
- Okta auth: SSWS token in `Authorization: SSWS <token>` header
- Graph auth: OAuth2 client credentials → Bearer token (same caching pattern as CrowdStrike)
- Config property `casehub.soc.identity.provider` selects `okta` or `graph`

---

## Batch 1: Identity provider foundation and connector

### Task 1: IdentityProvider interface, IdentityResult, and provider implementations

**Files:**
- Create: `app/src/main/java/io/casehub/soc/connector/identity/IdentityProvider.java`
- Create: `app/src/main/java/io/casehub/soc/connector/identity/IdentityResult.java`
- Create: `app/src/main/java/io/casehub/soc/connector/identity/IdentityApiException.java`
- Create: `app/src/main/java/io/casehub/soc/connector/identity/OktaIdentityProvider.java`
- Create: `app/src/main/java/io/casehub/soc/connector/identity/GraphIdentityProvider.java`
- Test: `app/src/test/java/io/casehub/soc/connector/identity/IdentityProviderTest.java`

**Interfaces:**
- Consumes: nothing (standalone)
- Produces:
  - `IdentityProvider` interface: `IdentityResult disableAccount(String userId)`, `IdentityResult revokeSessions(String userId)`, `IdentityResult rotateApiKey(String appId)`, `IdentityResult healthCheck()`, `String providerName()`
  - `static IdentityProvider unconfigured()` — factory for unconfigured state
  - `IdentityResult` record: `(boolean success, String details, String errorReason, boolean retryable, Map<String, Object> metadata)`
  - `IdentityApiException` extends `RuntimeException` with `int statusCode` field

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.soc.connector.identity;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class IdentityProviderTest {

    // --- Unconfigured provider ---

    @Test
    void unconfiguredProviderThrowsOnDisableAccount() {
        var provider = IdentityProvider.unconfigured();

        assertThatThrownBy(() -> provider.disableAccount("user-1"))
                .isInstanceOf(IdentityApiException.class)
                .hasMessageContaining("not configured");
    }

    @Test
    void unconfiguredProviderThrowsOnRevokeSessions() {
        var provider = IdentityProvider.unconfigured();

        assertThatThrownBy(() -> provider.revokeSessions("user-1"))
                .isInstanceOf(IdentityApiException.class)
                .hasMessageContaining("not configured");
    }

    @Test
    void unconfiguredProviderThrowsOnRotateApiKey() {
        var provider = IdentityProvider.unconfigured();

        assertThatThrownBy(() -> provider.rotateApiKey("app-1"))
                .isInstanceOf(IdentityApiException.class)
                .hasMessageContaining("not configured");
    }

    @Test
    void unconfiguredProviderName() {
        assertThat(IdentityProvider.unconfigured().providerName()).isEqualTo("unconfigured");
    }

    // --- Okta provider construction ---

    @Test
    void oktaProviderReportsName() {
        var provider = new OktaIdentityProvider("https://dev.okta.com", "ssws-token");
        assertThat(provider.providerName()).isEqualTo("okta");
    }

    @Test
    void oktaProviderBuildsDisableUrl() {
        var provider = new OktaIdentityProvider("https://dev.okta.com", "ssws-token");
        assertThat(provider.buildDisableUrl("user-123"))
                .isEqualTo("https://dev.okta.com/api/v1/users/user-123/lifecycle/suspend");
    }

    @Test
    void oktaProviderBuildsRevokeUrl() {
        var provider = new OktaIdentityProvider("https://dev.okta.com", "ssws-token");
        assertThat(provider.buildRevokeUrl("user-123"))
                .isEqualTo("https://dev.okta.com/api/v1/users/user-123/sessions");
    }

    @Test
    void oktaProviderBuildsRotateUrl() {
        var provider = new OktaIdentityProvider("https://dev.okta.com", "ssws-token");
        assertThat(provider.buildRotateUrl("app-456"))
                .isEqualTo("https://dev.okta.com/api/v1/apps/app-456/credentials/keys/generate?validityYears=1");
    }

    // --- Graph provider construction ---

    @Test
    void graphProviderReportsName() {
        var provider = new GraphIdentityProvider(
                "https://graph.microsoft.com", "tenant-1", "client-1", "secret-1");
        assertThat(provider.providerName()).isEqualTo("graph");
    }

    @Test
    void graphProviderBuildsDisableUrl() {
        var provider = new GraphIdentityProvider(
                "https://graph.microsoft.com", "tenant-1", "client-1", "secret-1");
        assertThat(provider.buildDisableUrl("user-123"))
                .isEqualTo("https://graph.microsoft.com/v1.0/users/user-123");
    }

    @Test
    void graphProviderBuildsRevokeUrl() {
        var provider = new GraphIdentityProvider(
                "https://graph.microsoft.com", "tenant-1", "client-1", "secret-1");
        assertThat(provider.buildRevokeUrl("user-123"))
                .isEqualTo("https://graph.microsoft.com/v1.0/users/user-123/revokeSignInSessions");
    }

    @Test
    void graphProviderBuildsRotateUrl() {
        var provider = new GraphIdentityProvider(
                "https://graph.microsoft.com", "tenant-1", "client-1", "secret-1");
        assertThat(provider.buildRotateUrl("app-456"))
                .isEqualTo("https://graph.microsoft.com/v1.0/applications/app-456/addPassword");
    }

    @Test
    void graphProviderBuildsTokenUrl() {
        var provider = new GraphIdentityProvider(
                "https://graph.microsoft.com", "tenant-1", "client-1", "secret-1");
        assertThat(provider.buildTokenUrl())
                .isEqualTo("https://login.microsoftonline.com/tenant-1/oauth2/v2.0/token");
    }

    // --- IdentityResult ---

    @Test
    void successResultIsSuccess() {
        var result = IdentityResult.success("done", java.util.Map.of("key", "val"));
        assertThat(result.success()).isTrue();
        assertThat(result.details()).isEqualTo("done");
        assertThat(result.errorReason()).isNull();
        assertThat(result.retryable()).isFalse();
    }

    @Test
    void failureResultIsNotSuccess() {
        var result = IdentityResult.failure("bad", true);
        assertThat(result.success()).isFalse();
        assertThat(result.errorReason()).isEqualTo("bad");
        assertThat(result.retryable()).isTrue();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=IdentityProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — classes do not exist

- [ ] **Step 3: Implement IdentityResult**

```java
package io.casehub.soc.connector.identity;

import java.util.Map;

public record IdentityResult(boolean success, String details, String errorReason,
                               boolean retryable, Map<String, Object> metadata) {

    public static IdentityResult success(String details, Map<String, Object> metadata) {
        return new IdentityResult(true, details, null, false, metadata);
    }

    public static IdentityResult failure(String errorReason, boolean retryable) {
        return new IdentityResult(false, null, errorReason, retryable, Map.of());
    }

    public static IdentityResult failure(String errorReason, boolean retryable,
                                          Map<String, Object> metadata) {
        return new IdentityResult(false, null, errorReason, retryable, metadata);
    }
}
```

- [ ] **Step 4: Implement IdentityApiException**

```java
package io.casehub.soc.connector.identity;

public class IdentityApiException extends RuntimeException {
    private final int statusCode;

    public IdentityApiException(String message) {
        super(message);
        this.statusCode = -1;
    }

    public IdentityApiException(String message, int statusCode) {
        super(message);
        this.statusCode = statusCode;
    }

    public IdentityApiException(String message, Throwable cause) {
        super(message, cause);
        this.statusCode = -1;
    }

    public int statusCode() {
        return statusCode;
    }
}
```

- [ ] **Step 5: Implement IdentityProvider interface**

```java
package io.casehub.soc.connector.identity;

public interface IdentityProvider {

    IdentityResult disableAccount(String userId);

    IdentityResult revokeSessions(String userId);

    IdentityResult rotateApiKey(String appId);

    IdentityResult healthCheck();

    String providerName();

    static IdentityProvider unconfigured() {
        return new IdentityProvider() {
            @Override public IdentityResult disableAccount(String userId) {
                throw new IdentityApiException("Identity provider not configured");
            }
            @Override public IdentityResult revokeSessions(String userId) {
                throw new IdentityApiException("Identity provider not configured");
            }
            @Override public IdentityResult rotateApiKey(String appId) {
                throw new IdentityApiException("Identity provider not configured");
            }
            @Override public IdentityResult healthCheck() {
                throw new IdentityApiException("Identity provider not configured");
            }
            @Override public String providerName() { return "unconfigured"; }
        };
    }
}
```

- [ ] **Step 6: Implement OktaIdentityProvider**

```java
package io.casehub.soc.connector.identity;

import io.vertx.core.Vertx;
import io.vertx.core.buffer.Buffer;
import io.vertx.ext.web.client.HttpResponse;
import io.vertx.ext.web.client.WebClient;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.concurrent.TimeUnit;

public class OktaIdentityProvider implements IdentityProvider {

    private static final Logger LOG = Logger.getLogger(OktaIdentityProvider.class);

    private final String apiBase;
    private final String apiToken;

    public OktaIdentityProvider(String apiBase, String apiToken) {
        this.apiBase = apiBase;
        this.apiToken = apiToken;
    }

    @Override
    public IdentityResult disableAccount(String userId) {
        String url = buildDisableUrl(userId);
        HttpResponse<Buffer> response = post(url, null);
        return handleResponse(response, "User " + userId + " disabled via okta",
                Map.of("provider", "okta", "identity_user_id", userId));
    }

    @Override
    public IdentityResult revokeSessions(String userId) {
        String url = buildRevokeUrl(userId);
        HttpResponse<Buffer> response = delete(url);
        return handleResponse(response, "Sessions revoked for " + userId + " via okta",
                Map.of("provider", "okta", "identity_user_id", userId));
    }

    @Override
    public IdentityResult rotateApiKey(String appId) {
        String url = buildRotateUrl(appId);
        HttpResponse<Buffer> response = post(url, null);
        String keyId = "";
        if (response.statusCode() >= 200 && response.statusCode() < 300
                && response.bodyAsString() != null) {
            try {
                var json = response.bodyAsJsonObject();
                keyId = json.getString("kid", "");
            } catch (Exception ignored) {}
        }
        return handleResponse(response, "API key rotated for " + appId + " via okta",
                Map.of("provider", "okta", "identity_key_id", keyId));
    }

    @Override
    public IdentityResult healthCheck() {
        String url = apiBase + "/api/v1/org";
        try {
            HttpResponse<Buffer> response = get(url);
            if (response.statusCode() >= 200 && response.statusCode() < 300) {
                return IdentityResult.success("okta reachable", Map.of("provider", "okta"));
            }
            return IdentityResult.failure("okta health check failed: HTTP " + response.statusCode(), false);
        } catch (Exception e) {
            return IdentityResult.failure("okta unreachable: " + e.getMessage(), true);
        }
    }

    @Override
    public String providerName() {
        return "okta";
    }

    String buildDisableUrl(String userId) {
        return apiBase + "/api/v1/users/" + userId + "/lifecycle/suspend";
    }

    String buildRevokeUrl(String userId) {
        return apiBase + "/api/v1/users/" + userId + "/sessions";
    }

    String buildRotateUrl(String appId) {
        return apiBase + "/api/v1/apps/" + appId + "/credentials/keys/generate?validityYears=1";
    }

    private IdentityResult handleResponse(HttpResponse<Buffer> response, String successMsg,
                                            Map<String, Object> metadata) {
        int status = response.statusCode();
        if (status >= 200 && status < 300) {
            return IdentityResult.success(successMsg, metadata);
        }
        if (status == 401) {
            throw new IdentityApiException("okta authentication failed", 401);
        }
        if (status == 404) {
            throw new IdentityApiException("okta: user/app not found", 404);
        }
        if (status == 429) {
            throw new IdentityApiException("okta rate limited", 429);
        }
        if (status >= 500) {
            throw new IdentityApiException("okta server error: " + status, status);
        }
        throw new IdentityApiException("okta error: " + status, status);
    }

    private HttpResponse<Buffer> post(String url, String body) {
        try {
            WebClient client = WebClient.create(Vertx.vertx());
            var req = client.postAbs(url)
                    .putHeader("Authorization", "SSWS " + apiToken)
                    .putHeader("Accept", "application/json");
            if (body != null) {
                req.putHeader("Content-Type", "application/json");
                return req.sendBuffer(Buffer.buffer(body))
                        .toCompletionStage().toCompletableFuture()
                        .get(30, TimeUnit.SECONDS);
            }
            return req.send().toCompletionStage().toCompletableFuture()
                    .get(30, TimeUnit.SECONDS);
        } catch (IdentityApiException e) {
            throw e;
        } catch (Exception e) {
            throw new IdentityApiException("okta unreachable: " + e.getMessage(), e);
        }
    }

    private HttpResponse<Buffer> delete(String url) {
        try {
            WebClient client = WebClient.create(Vertx.vertx());
            return client.deleteAbs(url)
                    .putHeader("Authorization", "SSWS " + apiToken)
                    .putHeader("Accept", "application/json")
                    .send().toCompletionStage().toCompletableFuture()
                    .get(30, TimeUnit.SECONDS);
        } catch (IdentityApiException e) {
            throw e;
        } catch (Exception e) {
            throw new IdentityApiException("okta unreachable: " + e.getMessage(), e);
        }
    }

    private HttpResponse<Buffer> get(String url) {
        try {
            WebClient client = WebClient.create(Vertx.vertx());
            return client.getAbs(url)
                    .putHeader("Authorization", "SSWS " + apiToken)
                    .putHeader("Accept", "application/json")
                    .send().toCompletionStage().toCompletableFuture()
                    .get(10, TimeUnit.SECONDS);
        } catch (Exception e) {
            throw new IdentityApiException("okta unreachable: " + e.getMessage(), e);
        }
    }
}
```

- [ ] **Step 7: Implement GraphIdentityProvider**

```java
package io.casehub.soc.connector.identity;

import io.vertx.core.Vertx;
import io.vertx.core.buffer.Buffer;
import io.vertx.ext.web.client.HttpResponse;
import io.vertx.ext.web.client.WebClient;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class GraphIdentityProvider implements IdentityProvider {

    private static final Logger LOG = Logger.getLogger(GraphIdentityProvider.class);
    private static final Duration EXPIRY_BUFFER = Duration.ofSeconds(60);

    private final String apiBase;
    private final String tenantId;
    private final String clientId;
    private final String clientSecret;

    private volatile String cachedToken;
    private volatile Instant tokenExpiry = Instant.EPOCH;

    public GraphIdentityProvider(String apiBase, String tenantId,
                                  String clientId, String clientSecret) {
        this.apiBase = apiBase;
        this.tenantId = tenantId;
        this.clientId = clientId;
        this.clientSecret = clientSecret;
    }

    @Override
    public IdentityResult disableAccount(String userId) {
        String token = getAccessToken();
        String url = buildDisableUrl(userId);
        HttpResponse<Buffer> response = patch(url, token,
                "{\"accountEnabled\":false}");
        return handleResponse(response, "User " + userId + " disabled via graph",
                Map.of("provider", "graph", "identity_user_id", userId));
    }

    @Override
    public IdentityResult revokeSessions(String userId) {
        String token = getAccessToken();
        String url = buildRevokeUrl(userId);
        HttpResponse<Buffer> response = post(url, token, null);
        return handleResponse(response, "Sessions revoked for " + userId + " via graph",
                Map.of("provider", "graph", "identity_user_id", userId));
    }

    @Override
    public IdentityResult rotateApiKey(String appId) {
        String token = getAccessToken();
        String url = buildRotateUrl(appId);
        HttpResponse<Buffer> response = post(url, token,
                "{\"passwordCredential\":{\"displayName\":\"casehub-rotated\"}}");
        String keyId = "";
        if (response.statusCode() >= 200 && response.statusCode() < 300
                && response.bodyAsString() != null) {
            try {
                var json = response.bodyAsJsonObject();
                keyId = json.getString("keyId", "");
            } catch (Exception ignored) {}
        }
        return handleResponse(response, "API key rotated for " + appId + " via graph",
                Map.of("provider", "graph", "identity_key_id", keyId));
    }

    @Override
    public IdentityResult healthCheck() {
        try {
            getAccessToken();
            return IdentityResult.success("graph reachable", Map.of("provider", "graph"));
        } catch (Exception e) {
            return IdentityResult.failure("graph unreachable: " + e.getMessage(), true);
        }
    }

    @Override
    public String providerName() {
        return "graph";
    }

    String buildDisableUrl(String userId) {
        return apiBase + "/v1.0/users/" + userId;
    }

    String buildRevokeUrl(String userId) {
        return apiBase + "/v1.0/users/" + userId + "/revokeSignInSessions";
    }

    String buildRotateUrl(String appId) {
        return apiBase + "/v1.0/applications/" + appId + "/addPassword";
    }

    String buildTokenUrl() {
        return "https://login.microsoftonline.com/" + tenantId + "/oauth2/v2.0/token";
    }

    String getAccessToken() {
        if (cachedToken != null && Instant.now().plus(EXPIRY_BUFFER).isBefore(tokenExpiry)) {
            return cachedToken;
        }
        return refreshToken();
    }

    private String refreshToken() {
        String url = buildTokenUrl();
        String body = "client_id=" + clientId
                + "&client_secret=" + clientSecret
                + "&scope=https%3A%2F%2Fgraph.microsoft.com%2F.default"
                + "&grant_type=client_credentials";
        try {
            WebClient client = WebClient.create(Vertx.vertx());
            HttpResponse<Buffer> response = client
                    .postAbs(url)
                    .putHeader("Content-Type", "application/x-www-form-urlencoded")
                    .sendBuffer(Buffer.buffer(body))
                    .toCompletionStage().toCompletableFuture()
                    .get(10, TimeUnit.SECONDS);

            if (response.statusCode() != 200) {
                throw new IdentityApiException(
                        "graph OAuth2 token request failed: HTTP " + response.statusCode(),
                        response.statusCode());
            }

            var json = response.bodyAsJsonObject();
            String token = json.getString("access_token");
            int expiresIn = json.getInteger("expires_in", 3599);
            cachedToken = token;
            tokenExpiry = Instant.now().plus(Duration.ofSeconds(expiresIn));
            LOG.infof("Graph OAuth2 token refreshed (expires in %ds)", expiresIn);
            return token;
        } catch (IdentityApiException e) {
            throw e;
        } catch (Exception e) {
            throw new IdentityApiException(
                    "graph OAuth2 token request failed: " + e.getMessage(), e);
        }
    }

    void setCachedToken(String token, Instant expiry) {
        this.cachedToken = token;
        this.tokenExpiry = expiry;
    }

    private IdentityResult handleResponse(HttpResponse<Buffer> response, String successMsg,
                                            Map<String, Object> metadata) {
        int status = response.statusCode();
        if (status >= 200 && status < 300) {
            return IdentityResult.success(successMsg, metadata);
        }
        if (status == 401) {
            throw new IdentityApiException("graph authentication failed", 401);
        }
        if (status == 404) {
            throw new IdentityApiException("graph: user/app not found", 404);
        }
        if (status == 429) {
            throw new IdentityApiException("graph rate limited", 429);
        }
        if (status >= 500) {
            throw new IdentityApiException("graph server error: " + status, status);
        }
        throw new IdentityApiException("graph error: " + status, status);
    }

    private HttpResponse<Buffer> post(String url, String token, String body) {
        try {
            WebClient client = WebClient.create(Vertx.vertx());
            var req = client.postAbs(url)
                    .putHeader("Authorization", "Bearer " + token)
                    .putHeader("Accept", "application/json");
            if (body != null) {
                req.putHeader("Content-Type", "application/json");
                return req.sendBuffer(Buffer.buffer(body))
                        .toCompletionStage().toCompletableFuture()
                        .get(30, TimeUnit.SECONDS);
            }
            return req.send().toCompletionStage().toCompletableFuture()
                    .get(30, TimeUnit.SECONDS);
        } catch (IdentityApiException e) {
            throw e;
        } catch (Exception e) {
            throw new IdentityApiException("graph unreachable: " + e.getMessage(), e);
        }
    }

    private HttpResponse<Buffer> patch(String url, String token, String body) {
        try {
            WebClient client = WebClient.create(Vertx.vertx());
            return client.patchAbs(url)
                    .putHeader("Authorization", "Bearer " + token)
                    .putHeader("Content-Type", "application/json")
                    .putHeader("Accept", "application/json")
                    .sendBuffer(Buffer.buffer(body))
                    .toCompletionStage().toCompletableFuture()
                    .get(30, TimeUnit.SECONDS);
        } catch (IdentityApiException e) {
            throw e;
        } catch (Exception e) {
            throw new IdentityApiException("graph unreachable: " + e.getMessage(), e);
        }
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=IdentityProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 16 tests PASS

- [ ] **Step 9: Commit**

```bash
git add app/src/main/java/io/casehub/soc/connector/identity/ app/src/test/java/io/casehub/soc/connector/identity/
git commit -m "feat(#54): add IdentityProvider interface, Okta and Graph implementations

Refs #54"
```

---

### Task 2: IdentityContainmentConnector, health check, and CDI config

**Files:**
- Create: `app/src/main/java/io/casehub/soc/connector/identity/IdentityContainmentConnector.java`
- Create: `app/src/main/java/io/casehub/soc/connector/identity/IdentityHealthCheck.java`
- Create: `app/src/main/java/io/casehub/soc/connector/identity/IdentityConnectorConfig.java`
- Test: `app/src/test/java/io/casehub/soc/connector/identity/IdentityContainmentConnectorTest.java`

**Interfaces:**
- Consumes: `IdentityProvider` interface, `IdentityResult`, `IdentityApiException` from Task 1
- Produces:
  - JAX-RS `POST /identity/containment` → `ContainmentResponse`
  - JAX-RS `GET /identity/health` → health status JSON
  - `ContainmentResponse validateAndExtract(ContainmentRequest request)` — package-private

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.soc.connector.identity;

import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class IdentityContainmentConnectorTest {

    @Test
    void disableUserAccountIsSupportedAction() {
        var connector = new IdentityContainmentConnector();
        assertThat(connector.isSupportedAction("disable.user.account")).isTrue();
    }

    @Test
    void revokeCredentialsIsSupportedAction() {
        var connector = new IdentityContainmentConnector();
        assertThat(connector.isSupportedAction("revoke.credentials")).isTrue();
    }

    @Test
    void rotateApiKeyIsSupportedAction() {
        var connector = new IdentityContainmentConnector();
        assertThat(connector.isSupportedAction("rotate.api.key")).isTrue();
    }

    @Test
    void unsupportedActionTypeReturnsFailure() {
        var connector = new IdentityContainmentConnector();

        var request = new ContainmentRequest(
                "isolate.host", Map.of("userId", "u-1"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("Unsupported");
        assertThat(response.retryable()).isFalse();
    }

    @Test
    void disableAccountMissingUserIdReturnsFailure() {
        var connector = new IdentityContainmentConnector();

        var request = new ContainmentRequest(
                "disable.user.account", Map.of(),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("userId");
    }

    @Test
    void revokeCredentialsMissingUserIdReturnsFailure() {
        var connector = new IdentityContainmentConnector();

        var request = new ContainmentRequest(
                "revoke.credentials", Map.of(),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("userId");
    }

    @Test
    void rotateApiKeyMissingAppIdReturnsFailure() {
        var connector = new IdentityContainmentConnector();

        var request = new ContainmentRequest(
                "rotate.api.key", Map.of(),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("appId");
    }

    @Test
    void validDisableAccountPassesValidation() {
        var connector = new IdentityContainmentConnector();

        var request = new ContainmentRequest(
                "disable.user.account", Map.of("userId", "user-123"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        assertThat(connector.validateAndExtract(request)).isNull();
    }

    @Test
    void validRevokeCredentialsPassesValidation() {
        var connector = new IdentityContainmentConnector();

        var request = new ContainmentRequest(
                "revoke.credentials", Map.of("userId", "user-123"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        assertThat(connector.validateAndExtract(request)).isNull();
    }

    @Test
    void validRotateApiKeyPassesValidation() {
        var connector = new IdentityContainmentConnector();

        var request = new ContainmentRequest(
                "rotate.api.key", Map.of("appId", "app-456"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        assertThat(connector.validateAndExtract(request)).isNull();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=IdentityContainmentConnectorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — class does not exist

- [ ] **Step 3: Implement IdentityContainmentConnector**

```java
package io.casehub.soc.connector.identity;

import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.Set;

@Path("/identity/containment")
@ApplicationScoped
public class IdentityContainmentConnector {

    private static final Logger LOG = Logger.getLogger(IdentityContainmentConnector.class);

    private static final Set<String> SUPPORTED_ACTIONS =
            Set.of("disable.user.account", "revoke.credentials", "rotate.api.key");

    @Inject
    IdentityProvider provider;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public ContainmentResponse execute(ContainmentRequest request) {
        ContainmentResponse validation = validateAndExtract(request);
        if (validation != null) {
            return validation;
        }

        try {
            IdentityResult result = switch (request.actionType()) {
                case "disable.user.account" ->
                        provider.disableAccount(request.parameters().get("userId").toString());
                case "revoke.credentials" ->
                        provider.revokeSessions(request.parameters().get("userId").toString());
                case "rotate.api.key" ->
                        provider.rotateApiKey(request.parameters().get("appId").toString());
                default -> IdentityResult.failure(
                        "Unsupported identity action: " + request.actionType(), false);
            };

            return new ContainmentResponse(result.success(), result.details(),
                    result.errorReason(), result.retryable(), result.metadata());
        } catch (IdentityApiException e) {
            LOG.warnf("%s API failed for %s: %s",
                    provider.providerName(), request.actionType(), e.getMessage());
            boolean retryable = e.statusCode() == 429 || e.statusCode() >= 500
                    || e.getMessage().contains("unreachable")
                    || e.getMessage().contains("token request failed");
            return new ContainmentResponse(false, null,
                    e.getMessage(), retryable,
                    Map.of("provider", provider.providerName()));
        }
    }

    boolean isSupportedAction(String actionType) {
        return SUPPORTED_ACTIONS.contains(actionType);
    }

    ContainmentResponse validateAndExtract(ContainmentRequest request) {
        if (!isSupportedAction(request.actionType())) {
            return new ContainmentResponse(false, null,
                    "Unsupported identity action type: " + request.actionType(),
                    false, Map.of());
        }

        return switch (request.actionType()) {
            case "disable.user.account", "revoke.credentials" ->
                    requireParam(request, "userId");
            case "rotate.api.key" -> requireParam(request, "appId");
            default -> null;
        };
    }

    private ContainmentResponse requireParam(ContainmentRequest request, String param) {
        Object value = request.parameters().get(param);
        if (value == null || value.toString().isBlank()) {
            return new ContainmentResponse(false, null,
                    "Missing required parameter: " + param, false, Map.of());
        }
        return null;
    }
}
```

- [ ] **Step 4: Implement IdentityHealthCheck**

```java
package io.casehub.soc.connector.identity;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.Map;

@Path("/identity/health")
@ApplicationScoped
public class IdentityHealthCheck {

    @Inject
    IdentityProvider provider;

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response health() {
        try {
            IdentityResult result = provider.healthCheck();
            if (result.success()) {
                return Response.ok(Map.of("status", "UP",
                        "provider", provider.providerName())).build();
            }
            return Response.status(503)
                    .entity(Map.of("status", "DOWN",
                            "provider", provider.providerName(),
                            "reason", result.errorReason())).build();
        } catch (Exception e) {
            return Response.status(503)
                    .entity(Map.of("status", "DOWN", "reason", e.getMessage()))
                    .build();
        }
    }
}
```

- [ ] **Step 5: Implement IdentityConnectorConfig**

```java
package io.casehub.soc.connector.identity;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Singleton;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.Optional;

@ApplicationScoped
public class IdentityConnectorConfig {

    @Produces
    @Singleton
    public IdentityProvider identityProvider(
            @ConfigProperty(name = "casehub.soc.identity.provider",
                             defaultValue = "okta") String providerType,
            @ConfigProperty(name = "casehub.soc.identity.api-base",
                             defaultValue = "") String apiBase,
            @ConfigProperty(name = "casehub.soc.identity.api-token") Optional<String> apiToken,
            @ConfigProperty(name = "casehub.soc.identity.tenant-id") Optional<String> tenantId,
            @ConfigProperty(name = "casehub.soc.identity.client-id") Optional<String> clientId,
            @ConfigProperty(name = "casehub.soc.identity.client-secret") Optional<String> clientSecret) {

        return switch (providerType) {
            case "okta" -> {
                if (apiToken.isEmpty() || apiToken.get().isBlank()) {
                    yield IdentityProvider.unconfigured();
                }
                yield new OktaIdentityProvider(apiBase, apiToken.get());
            }
            case "graph" -> {
                if (clientId.isEmpty() || clientId.get().isBlank()
                        || clientSecret.isEmpty() || clientSecret.get().isBlank()
                        || tenantId.isEmpty() || tenantId.get().isBlank()) {
                    yield IdentityProvider.unconfigured();
                }
                yield new GraphIdentityProvider(
                        apiBase.isBlank() ? "https://graph.microsoft.com" : apiBase,
                        tenantId.get(), clientId.get(), clientSecret.get());
            }
            default -> IdentityProvider.unconfigured();
        };
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=IdentityContainmentConnectorTest,IdentityProviderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/soc/connector/identity/ app/src/test/java/io/casehub/soc/connector/identity/
git commit -m "feat(#54): add IdentityContainmentConnector, health check, and CDI config

Refs #54"
```

---

## Batch 2: WireMock contract tests and config

### Task 3: WireMock contract tests for both providers + application.properties config

**Files:**
- Create: `app/src/test/java/io/casehub/soc/connector/identity/IdentityContainmentWireMockTest.java`
- Modify: `app/src/main/resources/application.properties` (add identity connector config defaults)

**Interfaces:**
- Consumes: `IdentityContainmentConnector` (Task 2), `OktaIdentityProvider`, `GraphIdentityProvider` (Task 1)
- Produces: Contract test coverage, identity config properties

- [ ] **Step 1: Write WireMock contract test**

```java
package io.casehub.soc.connector.identity;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Nested;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

class IdentityContainmentWireMockTest {

    static WireMockServer wireMock;

    @BeforeAll
    static void startWireMock() {
        wireMock = new WireMockServer(0);
        wireMock.start();
        WireMock.configureFor(wireMock.port());
    }

    @AfterAll
    static void stopWireMock() {
        wireMock.stop();
    }

    @Nested
    class OktaProviderTests {

        IdentityContainmentConnector connector;

        @BeforeEach
        void setUp() {
            wireMock.resetAll();
            String baseUrl = "http://localhost:" + wireMock.port();
            var provider = new OktaIdentityProvider(baseUrl, "test-ssws-token");
            connector = new IdentityContainmentConnector();
            connector.provider = provider;
        }

        @Test
        void successfulAccountDisable() {
            stubFor(post(urlPathMatching("/api/v1/users/.*/lifecycle/suspend"))
                    .willReturn(okJson("{}")));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "user-123"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isTrue();
            assertThat(response.details()).contains("user-123");
            assertThat(response.details()).contains("okta");
            assertThat(response.metadata()).containsEntry("provider", "okta");

            verify(postRequestedFor(urlPathMatching("/api/v1/users/.*/lifecycle/suspend"))
                    .withHeader("Authorization", equalTo("SSWS test-ssws-token")));
        }

        @Test
        void successfulSessionRevocation() {
            stubFor(delete(urlPathMatching("/api/v1/users/.*/sessions"))
                    .willReturn(aResponse().withStatus(204)));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "revoke.credentials", Map.of("userId", "user-123"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isTrue();
            assertThat(response.details()).contains("revoked");
        }

        @Test
        void successfulApiKeyRotation() {
            stubFor(post(urlPathMatching("/api/v1/apps/.*/credentials/keys/generate.*"))
                    .willReturn(aResponse().withStatus(201)
                            .withHeader("Content-Type", "application/json")
                            .withBody("{\"kid\":\"new-key-id\",\"created\":\"2026-09-14\"}")));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "rotate.api.key", Map.of("appId", "app-456"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isTrue();
            assertThat(response.details()).contains("app-456");
        }

        @Test
        void userNotFoundReturnsFailure() {
            stubFor(post(urlPathMatching("/api/v1/users/.*/lifecycle/suspend"))
                    .willReturn(aResponse().withStatus(404)));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "unknown"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isFalse();
            assertThat(response.errorReason()).contains("not found");
            assertThat(response.retryable()).isFalse();
        }

        @Test
        void serverErrorReturnsRetryable() {
            stubFor(post(urlPathMatching("/api/v1/users/.*/lifecycle/suspend"))
                    .willReturn(aResponse().withStatus(500)));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "user-123"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isFalse();
            assertThat(response.retryable()).isTrue();
        }

        @Test
        void rateLimitedReturnsRetryable() {
            stubFor(post(urlPathMatching("/api/v1/users/.*/lifecycle/suspend"))
                    .willReturn(aResponse().withStatus(429)));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "user-123"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isFalse();
            assertThat(response.retryable()).isTrue();
        }
    }

    @Nested
    class GraphProviderTests {

        IdentityContainmentConnector connector;

        @BeforeEach
        void setUp() {
            wireMock.resetAll();
            String baseUrl = "http://localhost:" + wireMock.port();

            stubFor(post(urlPathMatching(".*/oauth2/v2.0/token"))
                    .willReturn(okJson("{\"access_token\":\"test-graph-token\",\"expires_in\":3599}")));

            var provider = new GraphIdentityProvider(
                    baseUrl, "test-tenant", "test-client", "test-secret");
            connector = new IdentityContainmentConnector();
            connector.provider = provider;
        }

        @Test
        void successfulAccountDisable() {
            stubFor(patch(urlPathMatching("/v1.0/users/.*"))
                    .willReturn(aResponse().withStatus(204)));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "user-789"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isTrue();
            assertThat(response.details()).contains("user-789");
            assertThat(response.details()).contains("graph");
            assertThat(response.metadata()).containsEntry("provider", "graph");

            verify(patchRequestedFor(urlPathMatching("/v1.0/users/.*"))
                    .withHeader("Authorization", equalTo("Bearer test-graph-token"))
                    .withRequestBody(containing("accountEnabled")));
        }

        @Test
        void successfulSessionRevocation() {
            stubFor(post(urlPathMatching("/v1.0/users/.*/revokeSignInSessions"))
                    .willReturn(okJson("{\"value\":true}")));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "revoke.credentials", Map.of("userId", "user-789"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isTrue();
            assertThat(response.details()).contains("revoked");
        }

        @Test
        void successfulApiKeyRotation() {
            stubFor(post(urlPathMatching("/v1.0/applications/.*/addPassword"))
                    .willReturn(okJson("{\"keyId\":\"new-graph-key\",\"secretText\":\"secret\"}")));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "rotate.api.key", Map.of("appId", "app-abc"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isTrue();
            assertThat(response.details()).contains("app-abc");
        }

        @Test
        void userNotFoundReturnsFailure() {
            stubFor(patch(urlPathMatching("/v1.0/users/.*"))
                    .willReturn(aResponse().withStatus(404)));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "unknown"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isFalse();
            assertThat(response.errorReason()).contains("not found");
            assertThat(response.retryable()).isFalse();
        }

        @Test
        void oAuth2TokenFailureReturnsRetryable() {
            wireMock.resetAll();
            stubFor(post(urlPathMatching(".*/oauth2/v2.0/token"))
                    .willReturn(aResponse().withStatus(401)
                            .withHeader("Content-Type", "application/json")
                            .withBody("{\"error\":\"invalid_client\"}")));

            var provider = new GraphIdentityProvider(
                    "http://localhost:" + wireMock.port(),
                    "test-tenant", "bad-client", "bad-secret");
            connector.provider = provider;

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "user-789"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isFalse();
            assertThat(response.errorReason()).contains("token request failed");
            assertThat(response.retryable()).isTrue();
        }

        @Test
        void serverErrorReturnsRetryable() {
            stubFor(patch(urlPathMatching("/v1.0/users/.*"))
                    .willReturn(aResponse().withStatus(500)));

            ContainmentResponse response = connector.execute(new ContainmentRequest(
                    "disable.user.account", Map.of("userId", "user-789"),
                    "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

            assertThat(response.success()).isFalse();
            assertThat(response.retryable()).isTrue();
        }
    }

    @Test
    void missingUserIdReturnsFailureWithoutHttpCall() {
        var connector = new IdentityContainmentConnector();
        connector.provider = IdentityProvider.unconfigured();

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "disable.user.account", Map.of(),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("userId");
        assertThat(response.retryable()).isFalse();

        wireMock.verify(0, anyRequestedFor(anyUrl()));
    }

    @Test
    void unsupportedActionReturnsFailureWithoutHttpCall() {
        var connector = new IdentityContainmentConnector();
        connector.provider = IdentityProvider.unconfigured();

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "isolate.host", Map.of("userId", "u-1"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L));

        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("Unsupported");

        wireMock.verify(0, anyRequestedFor(anyUrl()));
    }
}
```

- [ ] **Step 2: Run WireMock tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=IdentityContainmentWireMockTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 3: Add identity connector config to application.properties**

Append to `app/src/main/resources/application.properties`:

```properties
# Identity containment connector (Okta/Graph — credentials from env vars in production)
casehub.soc.identity.provider=okta
casehub.soc.identity.api-base=https://dev-123456.okta.com
casehub.soc.identity.api-token=${OKTA_API_TOKEN:}
casehub.soc.identity.tenant-id=${AZURE_TENANT_ID:}
casehub.soc.identity.client-id=${AZURE_CLIENT_ID:}
casehub.soc.identity.client-secret=${AZURE_CLIENT_SECRET:}
```

- [ ] **Step 4: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 5: Commit**

```bash
git add app/src/test/java/io/casehub/soc/connector/identity/ app/src/main/resources/application.properties
git commit -m "test(#54): add identity WireMock contract tests and config defaults

Refs #54"
```

---

## References

- `specs/issue-34-fix-upstream-api-breaks/2026-09-14-identity-containment-connector-design.md` — design spec
- `specs/issue-50-connector-containment-runtimes/2026-09-14-connector-containment-runtimes-design.md` — parent spec (Batch 4)
- `CrowdStrikeContainmentConnector.java:28` — reference connector
- `CrowdStrikeOAuth2Client.java:13` — reference OAuth2 token caching (reused by GraphIdentityProvider)
- `CrowdStrikeConfig.java:9` — reference CDI producer with Optional
- `PaloAltoContainmentConnector.java` — reference connector (action mapping)
- `PaloAltoConfig.java` — reference CDI producer
- `CrowdStrikeContainmentWireMockTest.java:17` — reference WireMock test
- `ContainmentRequest.java:5` — API contract (request)
- `ContainmentResponse.java:6` — API contract (response)
- `HttpContainmentExecutor.java:25` — executor that routes to connectors
- GitHub #54 — focal issue
- GitHub #50 — parent epic
