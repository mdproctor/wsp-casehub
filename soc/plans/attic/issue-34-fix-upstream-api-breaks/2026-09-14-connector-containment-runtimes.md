# Connector-Based Containment Runtimes — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #50 — Epic: Connector-based containment runtimes — HTTP/MCP workers for EDR and firewall APIs
**Issue group:** #50

**Goal:** Build HttpContainmentExecutor that routes containment actions to external HTTP connectors via config-driven routing, with a simulated connector for dev/test.

**Architecture:** `HttpContainmentExecutor` (`@ApplicationScoped`) displaces `LoggingContainmentExecutor` (`@DefaultBean`) via CDI. It reads action-type-to-endpoint-tag routing from Quarkus config, resolves endpoint URLs via `ContainmentEndpointResolver`, and makes synchronous HTTP POST calls to sidecar connectors using Vert.x WebClient. A `SimulatedContainmentConnector` JAX-RS endpoint provides a dev/test connector implementing the same API contract.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x WebClient, Jackson, JUnit 5, AssertJ

## Global Constraints

- No dependency on `casehub-workers-http` — all components are self-contained in SOC
- Config keys use hyphens (`isolate-host`) not dots — converted to dot format at `@PostConstruct`
- `ContainmentRequest`/`ContainmentResponse` records live in `api/` (pure Java, no Quarkus)
- All other new components live in `app/`
- Vert.x WebClient available via transitive `quarkus-vertx` dependency — no new pom.xml entries
- `@ApplicationScoped` on `HttpContainmentExecutor` displaces `@DefaultBean LoggingContainmentExecutor` — no `@Alternative` or `@Priority` needed

---

## Batch 1: API contract and endpoint resolver

### Task 1: ContainmentRequest and ContainmentResponse records

**Files:**
- Create: `api/src/main/java/io/casehub/soc/engine/spi/ContainmentRequest.java`
- Create: `api/src/main/java/io/casehub/soc/engine/spi/ContainmentResponse.java`
- Test: `api/src/test/java/io/casehub/soc/engine/spi/ContainmentContractTest.java`

**Interfaces:**
- Consumes: `ContainmentContext` (existing record in same package)
- Produces: `ContainmentRequest(String actionType, Map<String,Object> parameters, String caseId, String incidentId, String approver, String tenancyId, long timeoutMs)` — used by `HttpContainmentExecutor` (Task 3) and `SimulatedContainmentConnector` (Task 5)
- Produces: `ContainmentResponse(boolean success, String details, String errorReason, boolean retryable, Map<String,Object> metadata)` — returned by connectors, parsed by `HttpContainmentExecutor` (Task 3)

- [ ] **Step 1: Write the failing test for ContainmentRequest serialization**

```java
package io.casehub.soc.engine.spi;

import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class ContainmentContractTest {

    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void requestSerializationRoundTrip() throws Exception {
        var request = new ContainmentRequest(
                "isolate.host",
                Map.of("hostId", "srv-42"),
                UUID.randomUUID().toString(),
                "INC-001",
                "analyst-jane@corp.com",
                "tenant-1",
                30_000L);

        String json = mapper.writeValueAsString(request);
        var deserialized = mapper.readValue(json, ContainmentRequest.class);

        assertThat(deserialized.actionType()).isEqualTo("isolate.host");
        assertThat(deserialized.parameters()).containsEntry("hostId", "srv-42");
        assertThat(deserialized.approver()).isEqualTo("analyst-jane@corp.com");
        assertThat(deserialized.timeoutMs()).isEqualTo(30_000L);
    }

    @Test
    void responseSerializationRoundTrip() throws Exception {
        var response = new ContainmentResponse(
                true, "Host isolated", null, false,
                Map.of("crowdstrike_session_id", "cs-123"));

        String json = mapper.writeValueAsString(response);
        var deserialized = mapper.readValue(json, ContainmentResponse.class);

        assertThat(deserialized.success()).isTrue();
        assertThat(deserialized.details()).isEqualTo("Host isolated");
        assertThat(deserialized.metadata()).containsEntry("crowdstrike_session_id", "cs-123");
    }

    @Test
    void responseFailureRoundTrip() throws Exception {
        var response = new ContainmentResponse(
                false, null, "Connection refused", true, Map.of());

        String json = mapper.writeValueAsString(response);
        var deserialized = mapper.readValue(json, ContainmentResponse.class);

        assertThat(deserialized.success()).isFalse();
        assertThat(deserialized.errorReason()).isEqualTo("Connection refused");
        assertThat(deserialized.retryable()).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dtest=ContainmentContractTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `ContainmentRequest` and `ContainmentResponse` not found.

- [ ] **Step 3: Create ContainmentRequest record**

```java
package io.casehub.soc.engine.spi;

import java.util.Map;

public record ContainmentRequest(
    String actionType,
    Map<String, Object> parameters,
    String caseId,
    String incidentId,
    String approver,
    String tenancyId,
    long timeoutMs
) {
    public static ContainmentRequest from(String actionType, Map<String, Object> parameters,
                                           ContainmentContext context) {
        return new ContainmentRequest(
                actionType, parameters,
                context.caseId().toString(),
                context.incidentId(),
                context.approver(),
                context.tenancyId(),
                context.timeoutMs());
    }
}
```

- [ ] **Step 4: Create ContainmentResponse record**

```java
package io.casehub.soc.engine.spi;

import java.util.Map;

public record ContainmentResponse(
    boolean success,
    String details,
    String errorReason,
    boolean retryable,
    Map<String, Object> metadata
) {
    public ContainmentResult toResult() {
        if (success) {
            return ContainmentResult.success(details, java.time.Instant.now());
        }
        return ContainmentResult.failure(errorReason, retryable);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dtest=ContainmentContractTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 3 tests pass.

- [ ] **Step 6: Commit**

```bash
git add api/src/main/java/io/casehub/soc/engine/spi/ContainmentRequest.java api/src/main/java/io/casehub/soc/engine/spi/ContainmentResponse.java api/src/test/java/io/casehub/soc/engine/spi/ContainmentContractTest.java
git commit -m "feat(#50): add ContainmentRequest/ContainmentResponse API contract records

Refs #50"
```

### Task 2: ContainmentEndpointResolver and ContainmentEndpoint

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/ContainmentEndpoint.java`
- Create: `app/src/main/java/io/casehub/soc/engine/ContainmentEndpointResolver.java`
- Test: `app/src/test/java/io/casehub/soc/engine/ContainmentEndpointResolverTest.java`

**Interfaces:**
- Consumes: MicroProfile Config (`org.eclipse.microprofile.config.Config`)
- Produces: `ContainmentEndpoint(String url, String method, int timeoutSeconds)` — used by `HttpContainmentExecutor` (Task 3)
- Produces: `ContainmentEndpointResolver.resolve(String endpointTag) → ContainmentEndpoint` — used by `HttpContainmentExecutor` (Task 3)

- [ ] **Step 1: Write failing tests for ContainmentEndpointResolver**

```java
package io.casehub.soc.engine;

import org.eclipse.microprofile.config.Config;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ContainmentEndpointResolverTest {

    @Test
    void resolvesConfiguredEndpoint() {
        var resolver = new ContainmentEndpointResolver();
        resolver.initialize(Map.of(
                "casehub.soc.containment.endpoints.crowdstrike.url", "http://cs:8080/containment",
                "casehub.soc.containment.endpoints.crowdstrike.method", "POST",
                "casehub.soc.containment.endpoints.crowdstrike.timeout-seconds", "30"
        ));

        ContainmentEndpoint ep = resolver.resolve("crowdstrike");
        assertThat(ep.url()).isEqualTo("http://cs:8080/containment");
        assertThat(ep.method()).isEqualTo("POST");
        assertThat(ep.timeoutSeconds()).isEqualTo(30);
    }

    @Test
    void usesDefaultsWhenMethodAndTimeoutMissing() {
        var resolver = new ContainmentEndpointResolver();
        resolver.initialize(Map.of(
                "casehub.soc.containment.endpoints.sim.url", "http://localhost:8080/sim/containment"
        ));

        ContainmentEndpoint ep = resolver.resolve("sim");
        assertThat(ep.method()).isEqualTo("POST");
        assertThat(ep.timeoutSeconds()).isEqualTo(30);
    }

    @Test
    void throwsOnUnknownTag() {
        var resolver = new ContainmentEndpointResolver();
        resolver.initialize(Map.of());

        assertThatThrownBy(() -> resolver.resolve("nonexistent"))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("nonexistent");
    }

    @Test
    void resolvesMultipleEndpoints() {
        var resolver = new ContainmentEndpointResolver();
        resolver.initialize(Map.of(
                "casehub.soc.containment.endpoints.crowdstrike.url", "http://cs:8080/containment",
                "casehub.soc.containment.endpoints.paloalto.url", "http://pa:8080/containment",
                "casehub.soc.containment.endpoints.paloalto.timeout-seconds", "15"
        ));

        assertThat(resolver.resolve("crowdstrike").url()).isEqualTo("http://cs:8080/containment");
        assertThat(resolver.resolve("paloalto").timeoutSeconds()).isEqualTo(15);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ContainmentEndpointResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — classes not found.

- [ ] **Step 3: Create ContainmentEndpoint record**

```java
package io.casehub.soc.engine;

public record ContainmentEndpoint(String url, String method, int timeoutSeconds) {
    public static final String DEFAULT_METHOD = "POST";
    public static final int DEFAULT_TIMEOUT_SECONDS = 30;
}
```

- [ ] **Step 4: Create ContainmentEndpointResolver**

```java
package io.casehub.soc.engine;

import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.Config;

import java.util.HashMap;
import java.util.LinkedHashMap;
import java.util.Map;

@ApplicationScoped
public class ContainmentEndpointResolver {

    private static final String PREFIX = "casehub.soc.containment.endpoints.";

    @Inject
    Config config;

    private final Map<String, ContainmentEndpoint> endpoints = new HashMap<>();

    @PostConstruct
    void init() {
        if (config != null) {
            initialize(loadFromConfig());
        }
    }

    void initialize(Map<String, String> properties) {
        endpoints.clear();
        Map<String, Map<String, String>> grouped = new LinkedHashMap<>();

        properties.forEach((key, value) -> {
            if (key.startsWith(PREFIX)) {
                String remainder = key.substring(PREFIX.length());
                int dot = remainder.indexOf('.');
                if (dot > 0) {
                    String tag = remainder.substring(0, dot);
                    String prop = remainder.substring(dot + 1);
                    grouped.computeIfAbsent(tag, k -> new LinkedHashMap<>()).put(prop, value);
                }
            }
        });

        grouped.forEach((tag, props) -> {
            String url = props.get("url");
            if (url != null && !url.isBlank()) {
                String method = props.getOrDefault("method", ContainmentEndpoint.DEFAULT_METHOD);
                int timeout = parseTimeout(props.get("timeout-seconds"));
                endpoints.put(tag, new ContainmentEndpoint(url, method, timeout));
            }
        });
    }

    public ContainmentEndpoint resolve(String endpointTag) {
        ContainmentEndpoint ep = endpoints.get(endpointTag);
        if (ep == null) {
            throw new IllegalStateException(
                    "No containment endpoint configured for tag: " + endpointTag
                    + ". Add casehub.soc.containment.endpoints." + endpointTag + ".url to config.");
        }
        return ep;
    }

    private Map<String, String> loadFromConfig() {
        Map<String, String> result = new LinkedHashMap<>();
        for (String key : config.getPropertyNames()) {
            if (key.startsWith(PREFIX)) {
                config.getOptionalValue(key, String.class)
                        .ifPresent(value -> result.put(key, value));
            }
        }
        return result;
    }

    private static int parseTimeout(String value) {
        if (value == null || value.isBlank()) {
            return ContainmentEndpoint.DEFAULT_TIMEOUT_SECONDS;
        }
        try {
            return Integer.parseInt(value);
        } catch (NumberFormatException e) {
            return ContainmentEndpoint.DEFAULT_TIMEOUT_SECONDS;
        }
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ContainmentEndpointResolverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 4 tests pass.

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/ContainmentEndpoint.java app/src/main/java/io/casehub/soc/engine/ContainmentEndpointResolver.java app/src/test/java/io/casehub/soc/engine/ContainmentEndpointResolverTest.java
git commit -m "feat(#50): add ContainmentEndpointResolver — config-driven endpoint resolution

Refs #50"
```

## Batch 2: HTTP executor and simulated connector

### Task 3: HttpContainmentExecutor

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/HttpContainmentExecutor.java`
- Test: `app/src/test/java/io/casehub/soc/engine/HttpContainmentExecutorTest.java`

**Interfaces:**
- Consumes: `ContainmentEndpointResolver.resolve(String) → ContainmentEndpoint` (Task 2)
- Consumes: `ContainmentRequest.from(String, Map, ContainmentContext)` (Task 1)
- Consumes: `ContainmentResponse.toResult()` (Task 1)
- Produces: implements `ContainmentExecutor.execute(String, Map, ContainmentContext) → ContainmentResult` — displaces `LoggingContainmentExecutor`

- [ ] **Step 1: Write failing tests for HttpContainmentExecutor**

Tests use a WireMock-style approach — start a local HTTP server, point the executor at it, verify behavior for all code paths.

```java
package io.casehub.soc.engine;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.soc.engine.spi.ContainmentContext;
import io.casehub.soc.engine.spi.ContainmentResult;
import io.vertx.core.Vertx;
import io.vertx.core.http.HttpServer;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;
import java.util.concurrent.atomic.AtomicReference;

import static org.assertj.core.api.Assertions.assertThat;

class HttpContainmentExecutorTest {

    private static Vertx vertx;
    private static HttpServer server;
    private static int port;
    private static final ObjectMapper MAPPER = new ObjectMapper();
    private static final AtomicReference<String> nextResponse = new AtomicReference<>();
    private static final AtomicReference<Integer> nextStatus = new AtomicReference<>(200);

    @BeforeAll
    static void startServer() throws Exception {
        vertx = Vertx.vertx();
        var latch = new CountDownLatch(1);
        server = vertx.createHttpServer()
                .requestHandler(req -> {
                    req.bodyHandler(body -> {
                        req.response()
                                .setStatusCode(nextStatus.get())
                                .putHeader("Content-Type", "application/json")
                                .end(nextResponse.get());
                    });
                })
                .listen(0, result -> {
                    port = result.result().actualPort();
                    latch.countDown();
                });
        latch.await(5, TimeUnit.SECONDS);
    }

    @AfterAll
    static void stopServer() {
        if (server != null) server.close();
        if (vertx != null) vertx.close();
    }

    private HttpContainmentExecutor buildExecutor(Map<String, String> routing) {
        var resolver = new ContainmentEndpointResolver();
        resolver.initialize(Map.of(
                "casehub.soc.containment.endpoints.test.url", "http://localhost:" + port + "/containment",
                "casehub.soc.containment.endpoints.test.timeout-seconds", "5"
        ));
        return new HttpContainmentExecutor(resolver, MAPPER, routing);
    }

    private ContainmentContext ctx() {
        return new ContainmentContext(UUID.randomUUID(), "INC-001", "analyst@corp.com", "tenant-1");
    }

    @Test
    void successfulExecution() {
        nextStatus.set(200);
        nextResponse.set("{\"success\":true,\"details\":\"Host isolated\",\"retryable\":false,\"metadata\":{}}");

        var executor = buildExecutor(Map.of("isolate.host", "test"));
        ContainmentResult result = executor.execute("isolate.host", Map.of("hostId", "srv-42"), ctx());

        assertThat(result.success()).isTrue();
        assertThat(result.details()).isEqualTo("Host isolated");
    }

    @Test
    void retryableFailure_serverError() {
        nextStatus.set(503);
        nextResponse.set("{\"error\":\"service unavailable\"}");

        var executor = buildExecutor(Map.of("isolate.host", "test"));
        ContainmentResult result = executor.execute("isolate.host", Map.of(), ctx());

        assertThat(result.success()).isFalse();
        assertThat(result.retryable()).isTrue();
    }

    @Test
    void permanentFailure_clientError() {
        nextStatus.set(400);
        nextResponse.set("{\"error\":\"bad request\"}");

        var executor = buildExecutor(Map.of("isolate.host", "test"));
        ContainmentResult result = executor.execute("isolate.host", Map.of(), ctx());

        assertThat(result.success()).isFalse();
        assertThat(result.retryable()).isFalse();
    }

    @Test
    void retryableFailure_429() {
        nextStatus.set(429);
        nextResponse.set("{\"error\":\"rate limited\"}");

        var executor = buildExecutor(Map.of("isolate.host", "test"));
        ContainmentResult result = executor.execute("isolate.host", Map.of(), ctx());

        assertThat(result.success()).isFalse();
        assertThat(result.retryable()).isTrue();
    }

    @Test
    void unmappedActionType_logsAndReturnsSuccess() {
        var executor = buildExecutor(Map.of());
        ContainmentResult result = executor.execute("enable.enhanced.logging", Map.of(), ctx());

        assertThat(result.success()).isTrue();
        assertThat(result.details()).contains("no connector");
    }

    @Test
    void malformedResponse_permanentFailure() {
        nextStatus.set(200);
        nextResponse.set("not json");

        var executor = buildExecutor(Map.of("isolate.host", "test"));
        ContainmentResult result = executor.execute("isolate.host", Map.of(), ctx());

        assertThat(result.success()).isFalse();
        assertThat(result.retryable()).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=HttpContainmentExecutorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — `HttpContainmentExecutor` not found.

- [ ] **Step 3: Implement HttpContainmentExecutor**

```java
package io.casehub.soc.engine;

import com.fasterxml.jackson.databind.ObjectMapper;
import io.casehub.soc.engine.spi.ContainmentContext;
import io.casehub.soc.engine.spi.ContainmentExecutor;
import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import io.casehub.soc.engine.spi.ContainmentResult;
import io.vertx.core.Vertx;
import io.vertx.core.buffer.Buffer;
import io.vertx.ext.web.client.HttpResponse;
import io.vertx.ext.web.client.WebClient;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.Config;
import org.jboss.logging.Logger;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;

@ApplicationScoped
public class HttpContainmentExecutor implements ContainmentExecutor {

    private static final Logger LOG = Logger.getLogger(HttpContainmentExecutor.class);
    private static final String ROUTING_PREFIX = "casehub.soc.containment.routing.";

    private final ContainmentEndpointResolver endpointResolver;
    private final ObjectMapper objectMapper;
    private final Map<String, String> actionRouting;
    private WebClient webClient;

    @Inject
    Config config;

    @Inject
    io.vertx.mutiny.core.Vertx mutinyVertx;

    HttpContainmentExecutor(ContainmentEndpointResolver endpointResolver,
                            ObjectMapper objectMapper,
                            Map<String, String> actionRouting) {
        this.endpointResolver = endpointResolver;
        this.objectMapper = objectMapper;
        this.actionRouting = new LinkedHashMap<>(actionRouting);
    }

    @Inject
    HttpContainmentExecutor(ContainmentEndpointResolver endpointResolver,
                            ObjectMapper objectMapper) {
        this.endpointResolver = endpointResolver;
        this.objectMapper = objectMapper;
        this.actionRouting = new LinkedHashMap<>();
    }

    @PostConstruct
    void init() {
        if (config != null) {
            for (String key : config.getPropertyNames()) {
                if (key.startsWith(ROUTING_PREFIX)) {
                    String hyphenated = key.substring(ROUTING_PREFIX.length());
                    String actionType = hyphenated.replace('-', '.');
                    config.getOptionalValue(key, String.class)
                            .ifPresent(tag -> actionRouting.put(actionType, tag));
                }
            }
        }
        if (mutinyVertx != null) {
            webClient = WebClient.create(Vertx.newInstance(mutinyVertx.getDelegate()));
        }
    }

    @Override
    public ContainmentResult execute(String actionType, Map<String, Object> parameters,
                                     ContainmentContext context) {
        String endpointTag = actionRouting.get(actionType);
        if (endpointTag == null) {
            LOG.infof("No routing for %s — logging only (no connector)", actionType);
            return ContainmentResult.success("Logged (no connector): " + actionType, Instant.now());
        }

        ContainmentEndpoint endpoint;
        try {
            endpoint = endpointResolver.resolve(endpointTag);
        } catch (IllegalStateException e) {
            return ContainmentResult.failure(e.getMessage(), false);
        }

        ContainmentRequest request = ContainmentRequest.from(actionType, parameters, context);

        try {
            String body = objectMapper.writeValueAsString(request);
            int timeoutMs = endpoint.timeoutSeconds() * 1000;

            WebClient client = webClient != null ? webClient
                    : WebClient.create(Vertx.vertx());

            HttpResponse<Buffer> response = client
                    .postAbs(endpoint.url())
                    .timeout(timeoutMs)
                    .putHeader("Content-Type", "application/json")
                    .sendBuffer(Buffer.buffer(body))
                    .toCompletionStage()
                    .toCompletableFuture()
                    .get(timeoutMs + 5000L, java.util.concurrent.TimeUnit.MILLISECONDS);

            return handleResponse(response);
        } catch (Exception e) {
            LOG.warnf("Containment HTTP call failed for %s → %s: %s",
                    actionType, endpointTag, e.getMessage());
            return ContainmentResult.failure("HTTP call failed: " + e.getMessage(), true);
        }
    }

    private ContainmentResult handleResponse(HttpResponse<Buffer> response) {
        int status = response.statusCode();

        if (status >= 200 && status < 300) {
            Buffer body = response.body();
            if (body == null || body.length() == 0) {
                return ContainmentResult.success("No response body", Instant.now());
            }
            try {
                ContainmentResponse cr = objectMapper.readValue(
                        body.getBytes(), ContainmentResponse.class);
                return cr.toResult();
            } catch (Exception e) {
                return ContainmentResult.failure(
                        "Malformed connector response: " + e.getMessage(), false);
            }
        }

        if (status == 429) {
            return ContainmentResult.failure("Rate limited (429)", true);
        }

        if (status >= 400 && status < 500) {
            return ContainmentResult.failure(status + " client error", false);
        }

        return ContainmentResult.failure(status + " server error", true);
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=HttpContainmentExecutorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 6 tests pass.

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/HttpContainmentExecutor.java app/src/test/java/io/casehub/soc/engine/HttpContainmentExecutorTest.java
git commit -m "feat(#50): add HttpContainmentExecutor — config-driven HTTP containment execution

Routes action types to external HTTP connectors via two-layer config.
Displaces LoggingContainmentExecutor via CDI @DefaultBean displacement.

Refs #50"
```

### Task 4: SimulatedContainmentConnector

**Files:**
- Create: `app/src/main/java/io/casehub/soc/rest/SimulatedContainmentConnector.java`
- Test: `app/src/test/java/io/casehub/soc/rest/SimulatedContainmentConnectorTest.java`

**Interfaces:**
- Consumes: `ContainmentRequest` (Task 1) — received as JSON POST body
- Produces: `ContainmentResponse` (Task 1) — returned as JSON response

- [ ] **Step 1: Write failing test for SimulatedContainmentConnector**

```java
package io.casehub.soc.rest;

import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class SimulatedContainmentConnectorTest {

    @Test
    void returnsSuccessForKnownActionType() {
        var connector = new SimulatedContainmentConnector();
        connector.init();

        var request = new ContainmentRequest(
                "isolate.host", Map.of("hostId", "srv-42"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.execute(request);

        assertThat(response.success()).isTrue();
        assertThat(response.details()).contains("isolate.host");
        assertThat(response.metadata()).isNotNull();
    }

    @Test
    void returnsSuccessForUnknownActionType() {
        var connector = new SimulatedContainmentConnector();
        connector.init();

        var request = new ContainmentRequest(
                "custom.action", Map.of(),
                "case-1", "INC-001", null, "tenant-1", 30_000L);

        ContainmentResponse response = connector.execute(request);

        assertThat(response.success()).isTrue();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SimulatedContainmentConnectorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation error — class not found.

- [ ] **Step 3: Implement SimulatedContainmentConnector**

```java
package io.casehub.soc.rest;

import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import jakarta.annotation.PostConstruct;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.UUID;
import java.util.concurrent.ThreadLocalRandom;

@Path("/sim/containment")
@ApplicationScoped
public class SimulatedContainmentConnector {

    private static final Logger LOG = Logger.getLogger(SimulatedContainmentConnector.class);

    @PostConstruct
    void init() {}

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public ContainmentResponse execute(ContainmentRequest request) {
        LOG.infof("SIM containment: %s for case %s (tenant=%s)",
                request.actionType(), request.caseId(), request.tenancyId());

        String simId = "sim-" + UUID.randomUUID().toString().substring(0, 8);

        return new ContainmentResponse(
                true,
                "Simulated " + request.actionType() + " executed successfully",
                null,
                false,
                Map.of("sim_id", simId,
                       "sim_action_type", request.actionType(),
                       "sim_latency_ms", ThreadLocalRandom.current().nextInt(50, 500)));
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SimulatedContainmentConnectorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 2 tests pass.

- [ ] **Step 5: Add config properties for dev/test routing**

Append to `app/src/main/resources/application.properties`:

```properties
# Containment connector routing (dev/test — all actions → simulated connector)
casehub.soc.containment.routing.isolate-host=sim
casehub.soc.containment.routing.wipe-endpoint=sim
casehub.soc.containment.routing.block-ip=sim
casehub.soc.containment.routing.block-domain=sim
casehub.soc.containment.routing.network-segmentation=sim
casehub.soc.containment.routing.disable-user-account=sim
casehub.soc.containment.routing.revoke-credentials=sim
casehub.soc.containment.routing.rotate-api-key=sim

# Simulated containment connector endpoint
casehub.soc.containment.endpoints.sim.url=http://localhost:${quarkus.http.port}/sim/containment
casehub.soc.containment.endpoints.sim.method=POST
casehub.soc.containment.endpoints.sim.timeout-seconds=10
```

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/soc/rest/SimulatedContainmentConnector.java app/src/test/java/io/casehub/soc/rest/SimulatedContainmentConnectorTest.java app/src/main/resources/application.properties
git commit -m "feat(#50): add SimulatedContainmentConnector and dev/test config

JAX-RS endpoint at /sim/containment implementing the containment API
contract. All action types routed to sim connector in dev/test profile.

Refs #50"
```

## Batch 3: Worker update and integration test

### Task 5: RuleContainmentExecutionWorker retryable failure path

**Files:**
- Modify: `app/src/main/java/io/casehub/soc/worker/RuleContainmentExecutionWorker.java`
- Modify: `app/src/test/java/io/casehub/soc/worker/RuleContainmentExecutionWorkerTest.java`

**Interfaces:**
- Consumes: `ContainmentResult.success()`, `ContainmentResult.retryable()`, `ContainmentResult.errorReason()` (existing)
- Produces: `WorkerResult` with `executed=false`, `success=false`, `errorReason` when `ContainmentResult` indicates failure

- [ ] **Step 1: Write failing test for retryable failure path**

Add to `RuleContainmentExecutionWorkerTest.java`:

```java
@Test
void recordsRetryableFailure() {
    ContainmentExecutor failingExecutor = (actionType, params, ctx) ->
            ContainmentResult.failure("Connection refused", true);

    Worker worker = RuleContainmentExecutionWorker.create(failingExecutor);
    Map<String, Object> input = new LinkedHashMap<>();
    input.put("containmentRecommendation", Map.of(
            "recommendedAction", "ISOLATE_HOST",
            "riskScore", 0.95,
            "actionParameters", Map.of("hostId", "srv-42")));
    input.put("alert", Map.of("detectedAt", "2026-09-02T10:00:00Z"));

    var output = outputMap(invokeWorker(worker, input));
    assertThat(output).containsEntry("executed", false);
    assertThat(output).containsEntry("success", false);
    assertThat(output).containsEntry("errorReason", "Connection refused");
}

@Test
void recordsPermanentFailure() {
    ContainmentExecutor failingExecutor = (actionType, params, ctx) ->
            ContainmentResult.failure("400 bad request", false);

    Worker worker = RuleContainmentExecutionWorker.create(failingExecutor);
    Map<String, Object> input = new LinkedHashMap<>();
    input.put("containmentRecommendation", Map.of(
            "recommendedAction", "BLOCK_IP",
            "riskScore", 0.5,
            "actionParameters", Map.of("ip", "10.0.1.99")));

    var output = outputMap(invokeWorker(worker, input));
    assertThat(output).containsEntry("executed", false);
    assertThat(output).containsEntry("success", false);
    assertThat(output).containsEntry("errorReason", "400 bad request");
}
```

- [ ] **Step 2: Run tests to verify the new tests fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RuleContainmentExecutionWorkerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: new tests fail — current code unconditionally maps as `executed=true` when executor returns.

- [ ] **Step 3: Add failure handling to RuleContainmentExecutionWorker**

In `RuleContainmentExecutionWorker.java`, after the `executor.execute()` call (line 55), add a failure check before building the success output:

```java
ContainmentResult result = executor.execute(actionType, actionParams, context);

if (!result.success()) {
    var output = new LinkedHashMap<String, Object>();
    output.put("actionType", actionType);
    output.put("executed", false);
    output.put("success", false);
    output.put("details", result.details());
    output.put("errorReason", result.errorReason());
    output.put("executionTimestamp", Instant.now().toString());
    output.put("detectionToContainmentMs", 0L);
    return WorkerResult.of(output);
}
```

The existing success path remains unchanged below this block.

- [ ] **Step 4: Run all worker tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RuleContainmentExecutionWorkerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 6 tests pass (4 existing + 2 new).

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/soc/worker/RuleContainmentExecutionWorker.java app/src/test/java/io/casehub/soc/worker/RuleContainmentExecutionWorkerTest.java
git commit -m "feat(#50): add retryable failure path to RuleContainmentExecutionWorker

When ContainmentResult.success() is false, return WorkerResult with
executed=false and errorReason. Enables fault pipeline escalation.

Refs #50"
```

### Task 6: Connector containment integration test

**Files:**
- Create: `app/src/test/java/io/casehub/soc/integration/ConnectorContainmentIntegrationTest.java`

**Interfaces:**
- Consumes: `HttpContainmentExecutor` (Task 3) — CDI displacement in test profile
- Consumes: `SimulatedContainmentConnector` (Task 4) — running as JAX-RS endpoint
- Consumes: full containment pipeline (existing) — alert → triage → gate → execution

- [ ] **Step 1: Write the integration test**

Model on existing `ContainmentPipelineIntegrationTest`. The key difference: `HttpContainmentExecutor` displaces `LoggingContainmentExecutor`, and the simulated connector responds to the HTTP call.

```java
package io.casehub.soc.integration;

import io.casehub.soc.engine.HttpContainmentExecutor;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class ConnectorContainmentIntegrationTest {

    @Inject
    io.casehub.soc.engine.spi.ContainmentExecutor containmentExecutor;

    @Test
    void httpExecutorDisplacesLogging() {
        assertThat(containmentExecutor).isInstanceOf(HttpContainmentExecutor.class);
    }

    @Test
    void executesViaSimulatedConnector() {
        var context = new io.casehub.soc.engine.spi.ContainmentContext(
                java.util.UUID.randomUUID(), "INC-TEST", "test-analyst", "test-tenant");

        var result = containmentExecutor.execute(
                "isolate.host",
                java.util.Map.of("hostId", "test-host"),
                context);

        assertThat(result.success()).isTrue();
        assertThat(result.details()).contains("isolate.host");
    }

    @Test
    void unmappedActionFallsThrough() {
        var context = new io.casehub.soc.engine.spi.ContainmentContext(
                java.util.UUID.randomUUID(), "INC-TEST", null, "test-tenant");

        var result = containmentExecutor.execute(
                "enable.enhanced.logging",
                java.util.Map.of(),
                context);

        assertThat(result.success()).isTrue();
        assertThat(result.details()).contains("no connector");
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=ConnectorContainmentIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: all 3 tests pass — CDI selects `HttpContainmentExecutor`, config routes to simulated connector, HTTP round-trip succeeds.

- [ ] **Step 3: Run the full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: all tests pass, including existing `ContainmentPipelineIntegrationTest`.

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/soc/integration/ConnectorContainmentIntegrationTest.java
git commit -m "test(#50): add ConnectorContainmentIntegrationTest — CDI wiring and simulated connector

Verifies HttpContainmentExecutor displaces LoggingContainmentExecutor,
routes isolate.host to simulated connector, and unmapped actions fall through.

Refs #50"
```

---

## References

- `specs/issue-50-connector-containment-runtimes/2026-09-14-connector-containment-runtimes-design.md` — design spec this plan implements
- `ContainmentExecutor.java:5` — SPI interface
- `LoggingContainmentExecutor.java:15` — `@DefaultBean` to be displaced
- `RuleContainmentExecutionWorker.java:19` — worker getting failure path
- `RuleContainmentExecutionWorkerTest.java:15` — existing worker test to extend
- `ContainmentResult.java:5` — result record with success/failure factories
- `ContainmentContext.java:5` — context record
- `SocInvestigationCaseDescriptor.java:39` — worker registration (unchanged)
- `ContainmentPipelineIntegrationTest.java` — existing integration test pattern
- `HttpEndpointResolver.java:25` — reference for config loading pattern
- GitHub #50 — focal epic
