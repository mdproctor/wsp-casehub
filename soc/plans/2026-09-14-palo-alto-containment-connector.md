# Palo Alto PAN-OS Containment Connector Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #53 — Palo Alto containment connector — IP/domain blocking and network segmentation
**Issue group:** #34, #52, #53, #54

**Goal:** Build an in-process Palo Alto PAN-OS connector that receives `ContainmentRequest` via JAX-RS, translates to PAN-OS XML API calls (config set + commit), polls commit jobs to completion, and returns `ContainmentResponse`.

**Architecture:** JAX-RS endpoint at `/paloalto/containment` following the `CrowdStrikeContainmentConnector` pattern. API key authentication (simpler than OAuth2 — no token lifecycle). PAN-OS XML API uses a two-phase commit model: stage config changes, then commit. The connector auto-commits per action and polls the commit job to completion before returning. String templates for XML request/response handling. WireMock stubs for contract tests against the PAN-OS API.

**Tech Stack:** Java 21, Quarkus 3.32.2, Vert.x WebClient (already available), WireMock (already in test scope from #52), AssertJ

## Global Constraints

- Java 21 source on Java 26 JVM
- Quarkus 3.32.2
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
- Package: `io.casehub.soc.connector.paloalto`
- All commits reference `Refs #53`
- No new runtime dependencies — Vert.x WebClient is already transitive
- WireMock already in test scope (added in #52)
- PAN-OS XML API: all calls use `POST /api/?key=<apikey>&type=<type>&...`
- API responses are XML: `<response status="success|error" code="N">...</response>`

---

## Batch 1: PAN-OS API client and connector foundation

### Task 1: PaloAltoApiClient — XML API calls, commit, and job polling

**Files:**
- Create: `app/src/main/java/io/casehub/soc/connector/paloalto/PaloAltoApiClient.java`
- Create: `app/src/main/java/io/casehub/soc/connector/paloalto/PaloAltoApiResponse.java`
- Create: `app/src/main/java/io/casehub/soc/connector/paloalto/PaloAltoApiException.java`
- Test: `app/src/test/java/io/casehub/soc/connector/paloalto/PaloAltoApiClientTest.java`

**Interfaces:**
- Consumes: nothing (standalone)
- Produces:
  - `PaloAltoApiResponse setConfig(String xpath, String element)` — stages a config change
  - `String commit()` — commits candidate config, polls to completion, returns job ID
  - `PaloAltoApiResponse systemInfo()` — returns system info (for health check)
  - `static PaloAltoApiClient unconfigured()` — factory for unconfigured state
  - `PaloAltoApiResponse` record: `(String status, int code, String message)`
  - `PaloAltoApiException` extends `RuntimeException` with `int code` field

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.soc.connector.paloalto;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class PaloAltoApiClientTest {

    @Test
    void unconfiguredClientThrowsOnSetConfig() {
        var client = PaloAltoApiClient.unconfigured();

        assertThatThrownBy(() -> client.setConfig("/some/xpath", "<element/>"))
                .isInstanceOf(PaloAltoApiException.class)
                .hasMessageContaining("not configured");
    }

    @Test
    void unconfiguredClientThrowsOnCommit() {
        var client = PaloAltoApiClient.unconfigured();

        assertThatThrownBy(() -> client.commit())
                .isInstanceOf(PaloAltoApiException.class)
                .hasMessageContaining("not configured");
    }

    @Test
    void unconfiguredClientThrowsOnSystemInfo() {
        var client = PaloAltoApiClient.unconfigured();

        assertThatThrownBy(() -> client.systemInfo())
                .isInstanceOf(PaloAltoApiException.class)
                .hasMessageContaining("not configured");
    }

    @Test
    void parsesSuccessResponse() {
        String xml = "<response status=\"success\" code=\"20\"><msg>command succeeded</msg></response>";

        PaloAltoApiResponse response = PaloAltoApiResponse.parse(xml);

        assertThat(response.isSuccess()).isTrue();
        assertThat(response.code()).isEqualTo(20);
        assertThat(response.message()).isEqualTo("command succeeded");
    }

    @Test
    void parsesErrorResponse() {
        String xml = "<response status=\"error\" code=\"12\"><msg><line>Object not found</line></msg></response>";

        PaloAltoApiResponse response = PaloAltoApiResponse.parse(xml);

        assertThat(response.isSuccess()).isFalse();
        assertThat(response.code()).isEqualTo(12);
        assertThat(response.message()).isEqualTo("Object not found");
    }

    @Test
    void parsesCommitJobId() {
        String xml = "<response status=\"success\" code=\"19\"><result><msg><line>Commit job enqueued with jobid 42</line></msg><job>42</job></result></response>";

        String jobId = PaloAltoApiResponse.parseJobId(xml);

        assertThat(jobId).isEqualTo("42");
    }

    @Test
    void parsesJobStatusComplete() {
        String xml = "<response status=\"success\"><result><job><id>42</id><status>FIN</status><result>OK</result><progress>100</progress></job></result></response>";

        PaloAltoApiResponse.JobStatus status = PaloAltoApiResponse.parseJobStatus(xml);

        assertThat(status.isFinished()).isTrue();
        assertThat(status.isSuccess()).isTrue();
        assertThat(status.jobId()).isEqualTo("42");
    }

    @Test
    void parsesJobStatusInProgress() {
        String xml = "<response status=\"success\"><result><job><id>42</id><status>ACT</status><progress>50</progress></job></result></response>";

        PaloAltoApiResponse.JobStatus status = PaloAltoApiResponse.parseJobStatus(xml);

        assertThat(status.isFinished()).isFalse();
    }

    @Test
    void parsesJobStatusFailed() {
        String xml = "<response status=\"success\"><result><job><id>42</id><status>FIN</status><result>FAIL</result><details><line>Configuration commit failed</line></details></job></result></response>";

        PaloAltoApiResponse.JobStatus status = PaloAltoApiResponse.parseJobStatus(xml);

        assertThat(status.isFinished()).isTrue();
        assertThat(status.isSuccess()).isFalse();
        assertThat(status.details()).isEqualTo("Configuration commit failed");
    }

    @Test
    void configuredClientReportsApiBase() {
        var client = new PaloAltoApiClient(
                "https://fw.corp.local", "test-key", "vsys1",
                "localhost.localdomain", 2000, 60000);

        assertThat(client.isConfigured()).isTrue();
    }

    @Test
    void blankApiKeyCreatesUnconfigured() {
        var client = new PaloAltoApiClient(
                "https://fw.corp.local", "", "vsys1",
                "localhost.localdomain", 2000, 60000);

        assertThat(client.isConfigured()).isFalse();
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=PaloAltoApiClientTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — classes do not exist

- [ ] **Step 3: Implement PaloAltoApiResponse**

```java
package io.casehub.soc.connector.paloalto;

import java.util.regex.Matcher;
import java.util.regex.Pattern;

public record PaloAltoApiResponse(String status, int code, String message) {

    private static final Pattern STATUS_PATTERN =
            Pattern.compile("status=\"(\\w+)\"");
    private static final Pattern CODE_PATTERN =
            Pattern.compile("code=\"(\\d+)\"");
    private static final Pattern MSG_PATTERN =
            Pattern.compile("<msg>(?:<line>)?(.*?)(?:</line>)?</msg>", Pattern.DOTALL);
    private static final Pattern JOB_ID_PATTERN =
            Pattern.compile("<job>(\\d+)</job>");
    private static final Pattern JOB_STATUS_PATTERN =
            Pattern.compile("<status>(\\w+)</status>");
    private static final Pattern JOB_RESULT_PATTERN =
            Pattern.compile("<result>(\\w+)</result>");
    private static final Pattern JOB_PROGRESS_PATTERN =
            Pattern.compile("<progress>(\\d+)</progress>");
    private static final Pattern JOB_DETAILS_PATTERN =
            Pattern.compile("<details><line>(.*?)</line></details>", Pattern.DOTALL);

    public boolean isSuccess() {
        return "success".equals(status);
    }

    public static PaloAltoApiResponse parse(String xml) {
        String status = extractGroup(STATUS_PATTERN, xml, "unknown");
        int code = Integer.parseInt(extractGroup(CODE_PATTERN, xml, "0"));
        String message = extractGroup(MSG_PATTERN, xml, "");
        return new PaloAltoApiResponse(status, code, message);
    }

    public static String parseJobId(String xml) {
        return extractGroup(JOB_ID_PATTERN, xml, null);
    }

    public static JobStatus parseJobStatus(String xml) {
        String jobId = extractGroup(JOB_ID_PATTERN, xml, "");
        String status = extractGroup(JOB_STATUS_PATTERN, xml, "");
        String result = extractGroup(JOB_RESULT_PATTERN, xml, "");
        int progress = Integer.parseInt(extractGroup(JOB_PROGRESS_PATTERN, xml, "0"));
        String details = extractGroup(JOB_DETAILS_PATTERN, xml, "");
        return new JobStatus(jobId, status, result, progress, details);
    }

    private static String extractGroup(Pattern pattern, String xml, String defaultValue) {
        Matcher m = pattern.matcher(xml);
        return m.find() ? m.group(1) : defaultValue;
    }

    public record JobStatus(String jobId, String status, String result,
                             int progress, String details) {
        public boolean isFinished() {
            return "FIN".equals(status);
        }

        public boolean isSuccess() {
            return isFinished() && "OK".equals(result);
        }
    }
}
```

- [ ] **Step 4: Implement PaloAltoApiException**

```java
package io.casehub.soc.connector.paloalto;

public class PaloAltoApiException extends RuntimeException {
    private final int code;

    public PaloAltoApiException(String message) {
        super(message);
        this.code = -1;
    }

    public PaloAltoApiException(String message, int code) {
        super(message);
        this.code = code;
    }

    public PaloAltoApiException(String message, Throwable cause) {
        super(message, cause);
        this.code = -1;
    }

    public int code() {
        return code;
    }
}
```

- [ ] **Step 5: Implement PaloAltoApiClient**

```java
package io.casehub.soc.connector.paloalto;

import io.vertx.core.Vertx;
import io.vertx.core.buffer.Buffer;
import io.vertx.ext.web.client.HttpResponse;
import io.vertx.ext.web.client.WebClient;
import org.jboss.logging.Logger;

import java.net.URLEncoder;
import java.nio.charset.StandardCharsets;
import java.util.concurrent.TimeUnit;

public class PaloAltoApiClient {

    private static final Logger LOG = Logger.getLogger(PaloAltoApiClient.class);

    private final String apiBase;
    private final String apiKey;
    private final String vsys;
    private final String deviceName;
    private final long commitPollIntervalMs;
    private final long commitTimeoutMs;

    public PaloAltoApiClient(String apiBase, String apiKey, String vsys,
                              String deviceName, long commitPollIntervalMs,
                              long commitTimeoutMs) {
        this.apiBase = apiBase;
        this.apiKey = apiKey;
        this.vsys = vsys;
        this.deviceName = deviceName;
        this.commitPollIntervalMs = commitPollIntervalMs;
        this.commitTimeoutMs = commitTimeoutMs;
    }

    private PaloAltoApiClient() {
        this.apiBase = "";
        this.apiKey = "";
        this.vsys = "";
        this.deviceName = "";
        this.commitPollIntervalMs = 0;
        this.commitTimeoutMs = 0;
    }

    public static PaloAltoApiClient unconfigured() {
        return new PaloAltoApiClient();
    }

    public boolean isConfigured() {
        return apiKey != null && !apiKey.isBlank();
    }

    public String vsys() {
        return vsys;
    }

    public String deviceName() {
        return deviceName;
    }

    private void requireConfigured() {
        if (!isConfigured()) {
            throw new PaloAltoApiException(
                    "PAN-OS not configured — set api-key");
        }
    }

    public PaloAltoApiResponse setConfig(String xpath, String element) {
        requireConfigured();
        String url = apiBase + "/api/?key=" + enc(apiKey)
                + "&type=config&action=set"
                + "&xpath=" + enc(xpath)
                + "&element=" + enc(element);
        String xml = postAndRead(url, 30_000);
        PaloAltoApiResponse response = PaloAltoApiResponse.parse(xml);
        if (!response.isSuccess()) {
            throw new PaloAltoApiException(
                    "PAN-OS config error: " + response.message(), response.code());
        }
        return response;
    }

    public String commit() {
        requireConfigured();
        String commitUrl = apiBase + "/api/?key=" + enc(apiKey)
                + "&type=commit&cmd=" + enc("<commit></commit>");
        String commitXml = postAndRead(commitUrl, 30_000);
        String jobId = PaloAltoApiResponse.parseJobId(commitXml);
        if (jobId == null) {
            throw new PaloAltoApiException("PAN-OS commit did not return a job ID");
        }
        LOG.infof("PAN-OS commit started, job ID: %s", jobId);
        return pollCommitJob(jobId);
    }

    private String pollCommitJob(String jobId) {
        String pollUrl = apiBase + "/api/?key=" + enc(apiKey)
                + "&type=op&cmd=" + enc("<show><jobs><id>" + jobId + "</id></jobs></show>");
        long deadline = System.currentTimeMillis() + commitTimeoutMs;

        while (System.currentTimeMillis() < deadline) {
            String xml = postAndRead(pollUrl, 10_000);
            PaloAltoApiResponse.JobStatus status = PaloAltoApiResponse.parseJobStatus(xml);

            if (status.isFinished()) {
                if (status.isSuccess()) {
                    LOG.infof("PAN-OS commit job %s completed successfully", jobId);
                    return jobId;
                } else {
                    throw new PaloAltoApiException(
                            "PAN-OS commit failed: " + status.details());
                }
            }

            try {
                Thread.sleep(commitPollIntervalMs);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                throw new PaloAltoApiException("Commit polling interrupted", e);
            }
        }

        throw new PaloAltoApiException(
                "PAN-OS commit timed out after " + (commitTimeoutMs / 1000) + "s");
    }

    public PaloAltoApiResponse systemInfo() {
        requireConfigured();
        String url = apiBase + "/api/?key=" + enc(apiKey)
                + "&type=op&cmd=" + enc("<show><system><info></info></system></show>");
        String xml = postAndRead(url, 10_000);
        return PaloAltoApiResponse.parse(xml);
    }

    private String postAndRead(String url, long timeoutMs) {
        try {
            WebClient client = WebClient.create(Vertx.vertx());
            HttpResponse<Buffer> response = client
                    .postAbs(url)
                    .send()
                    .toCompletionStage()
                    .toCompletableFuture()
                    .get(timeoutMs, TimeUnit.MILLISECONDS);

            int status = response.statusCode();
            if (status == 403) {
                throw new PaloAltoApiException("PAN-OS authentication failed", 403);
            }
            if (status == 429) {
                throw new PaloAltoApiException("PAN-OS rate limited", 429);
            }
            if (status >= 500) {
                throw new PaloAltoApiException(
                        "PAN-OS server error: " + status, status);
            }
            if (status >= 400) {
                throw new PaloAltoApiException(
                        "PAN-OS error: " + status, status);
            }

            return response.bodyAsString();
        } catch (PaloAltoApiException e) {
            throw e;
        } catch (Exception e) {
            throw new PaloAltoApiException(
                    "PAN-OS unreachable: " + e.getMessage(), e);
        }
    }

    private static String enc(String value) {
        return URLEncoder.encode(value, StandardCharsets.UTF_8);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=PaloAltoApiClientTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: 11 tests PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/soc/connector/paloalto/ app/src/test/java/io/casehub/soc/connector/paloalto/
git commit -m "feat(#53): add PaloAltoApiClient — XML API calls, commit, and job polling

Refs #53"
```

---

### Task 2: PaloAltoContainmentConnector, health check, and CDI config

**Files:**
- Create: `app/src/main/java/io/casehub/soc/connector/paloalto/PaloAltoContainmentConnector.java`
- Create: `app/src/main/java/io/casehub/soc/connector/paloalto/PaloAltoHealthCheck.java`
- Create: `app/src/main/java/io/casehub/soc/connector/paloalto/PaloAltoConfig.java`
- Test: `app/src/test/java/io/casehub/soc/connector/paloalto/PaloAltoContainmentConnectorTest.java`

**Interfaces:**
- Consumes: `PaloAltoApiClient.setConfig(xpath, element)`, `PaloAltoApiClient.commit()`, `PaloAltoApiClient.systemInfo()` from Task 1
- Produces:
  - JAX-RS `POST /paloalto/containment` → `ContainmentResponse`
  - JAX-RS `GET /paloalto/health` → health status JSON
  - `String buildXpath(String... segments)` — package-private, builds PAN-OS config xpath
  - `ContainmentResponse validateAndExtract(ContainmentRequest request)` — package-private, validates params per action type

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.soc.connector.paloalto;

import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;

class PaloAltoContainmentConnectorTest {

    @Test
    void blockIpIsSupportedAction() {
        var connector = new PaloAltoContainmentConnector();
        assertThat(connector.isSupportedAction("block.ip")).isTrue();
    }

    @Test
    void blockDomainIsSupportedAction() {
        var connector = new PaloAltoContainmentConnector();
        assertThat(connector.isSupportedAction("block.domain")).isTrue();
    }

    @Test
    void networkSegmentationIsSupportedAction() {
        var connector = new PaloAltoContainmentConnector();
        assertThat(connector.isSupportedAction("network.segmentation")).isTrue();
    }

    @Test
    void unsupportedActionTypeReturnsFailure() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "isolate.host", Map.of("ip", "1.2.3.4"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("Unsupported");
        assertThat(response.retryable()).isFalse();
    }

    @Test
    void blockIpMissingIpReturnsFailure() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "block.ip", Map.of(),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("ip");
        assertThat(response.retryable()).isFalse();
    }

    @Test
    void blockDomainMissingDomainReturnsFailure() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "block.domain", Map.of(),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("domain");
        assertThat(response.retryable()).isFalse();
    }

    @Test
    void networkSegmentationMissingSourceZoneReturnsFailure() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "network.segmentation", Map.of("destZone", "dmz", "ruleName", "test-rule"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("sourceZone");
    }

    @Test
    void networkSegmentationMissingDestZoneReturnsFailure() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "network.segmentation", Map.of("sourceZone", "trust", "ruleName", "test-rule"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("destZone");
    }

    @Test
    void networkSegmentationMissingRuleNameReturnsFailure() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "network.segmentation", Map.of("sourceZone", "trust", "destZone", "dmz"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNotNull();
        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("ruleName");
    }

    @Test
    void validBlockIpRequestPassesValidation() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "block.ip", Map.of("ip", "192.168.1.100"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNull();
    }

    @Test
    void validBlockDomainRequestPassesValidation() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "block.domain", Map.of("domain", "evil.com"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNull();
    }

    @Test
    void validNetworkSegmentationRequestPassesValidation() {
        var connector = new PaloAltoContainmentConnector();

        var request = new ContainmentRequest(
                "network.segmentation",
                Map.of("sourceZone", "trust", "destZone", "dmz", "ruleName", "casehub-seg-001"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 30_000L);

        ContainmentResponse response = connector.validateAndExtract(request);

        assertThat(response).isNull();
    }

    @Test
    void sanitizesIpForAddressObjectName() {
        assertThat(PaloAltoContainmentConnector.sanitizeIp("192.168.1.100"))
                .isEqualTo("192-168-1-100");
    }

    @Test
    void buildsAddressXpath() {
        var connector = new PaloAltoContainmentConnector();
        connector.deviceName = "localhost.localdomain";
        connector.vsys = "vsys1";

        String xpath = connector.buildAddressXpath("casehub-192-168-1-100");

        assertThat(xpath).isEqualTo(
                "/config/devices/entry[@name='localhost.localdomain']"
                + "/vsys/entry[@name='vsys1']"
                + "/address/entry[@name='casehub-192-168-1-100']");
    }

    @Test
    void buildsAddressGroupMemberXpath() {
        var connector = new PaloAltoContainmentConnector();
        connector.deviceName = "localhost.localdomain";
        connector.vsys = "vsys1";

        String xpath = connector.buildAddressGroupMemberXpath("casehub-blocked-ips");

        assertThat(xpath).isEqualTo(
                "/config/devices/entry[@name='localhost.localdomain']"
                + "/vsys/entry[@name='vsys1']"
                + "/address-group/entry[@name='casehub-blocked-ips']/static");
    }

    @Test
    void buildsUrlCategoryMemberXpath() {
        var connector = new PaloAltoContainmentConnector();
        connector.deviceName = "localhost.localdomain";
        connector.vsys = "vsys1";

        String xpath = connector.buildUrlCategoryMemberXpath("casehub-blocked-domains");

        assertThat(xpath).isEqualTo(
                "/config/devices/entry[@name='localhost.localdomain']"
                + "/vsys/entry[@name='vsys1']"
                + "/profiles/custom-url-category/entry[@name='casehub-blocked-domains']/list");
    }

    @Test
    void buildsSecurityRuleXpath() {
        var connector = new PaloAltoContainmentConnector();
        connector.deviceName = "localhost.localdomain";
        connector.vsys = "vsys1";

        String xpath = connector.buildSecurityRuleXpath("casehub-seg-001");

        assertThat(xpath).isEqualTo(
                "/config/devices/entry[@name='localhost.localdomain']"
                + "/vsys/entry[@name='vsys1']"
                + "/rulebase/security/rules/entry[@name='casehub-seg-001']");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=PaloAltoContainmentConnectorTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: Compilation error — class does not exist

- [ ] **Step 3: Implement PaloAltoContainmentConnector**

```java
package io.casehub.soc.connector.paloalto;

import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.Consumes;
import jakarta.ws.rs.POST;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.Map;
import java.util.Set;

@Path("/paloalto/containment")
@ApplicationScoped
public class PaloAltoContainmentConnector {

    private static final Logger LOG = Logger.getLogger(PaloAltoContainmentConnector.class);

    private static final Set<String> SUPPORTED_ACTIONS =
            Set.of("block.ip", "block.domain", "network.segmentation");

    @Inject
    PaloAltoApiClient apiClient;

    @ConfigProperty(name = "casehub.soc.paloalto.device-name",
                     defaultValue = "localhost.localdomain")
    String deviceName;

    @ConfigProperty(name = "casehub.soc.paloalto.vsys",
                     defaultValue = "vsys1")
    String vsys;

    @ConfigProperty(name = "casehub.soc.paloalto.default-address-group",
                     defaultValue = "casehub-blocked-ips")
    String defaultAddressGroup;

    @ConfigProperty(name = "casehub.soc.paloalto.default-url-category",
                     defaultValue = "casehub-blocked-domains")
    String defaultUrlCategory;

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public ContainmentResponse execute(ContainmentRequest request) {
        ContainmentResponse validation = validateAndExtract(request);
        if (validation != null) {
            return validation;
        }

        try {
            return switch (request.actionType()) {
                case "block.ip" -> executeBlockIp(request);
                case "block.domain" -> executeBlockDomain(request);
                case "network.segmentation" -> executeNetworkSegmentation(request);
                default -> new ContainmentResponse(false, null,
                        "Unsupported PAN-OS action type: " + request.actionType(),
                        false, Map.of());
            };
        } catch (PaloAltoApiException e) {
            LOG.warnf("PAN-OS API failed for %s: %s", request.actionType(), e.getMessage());
            boolean retryable = e.code() == 429 || e.code() >= 500
                    || e.getMessage().contains("unreachable")
                    || e.getMessage().contains("timed out")
                    || e.getMessage().contains("commit failed");
            return new ContainmentResponse(false, null,
                    e.getMessage(), retryable,
                    e.code() > 0 ? Map.of("paloalto_code", e.code()) : Map.of());
        }
    }

    private ContainmentResponse executeBlockIp(ContainmentRequest request) {
        String ip = request.parameters().get("ip").toString();
        String addressGroup = paramOrDefault(request, "addressGroup", defaultAddressGroup);
        String sanitized = sanitizeIp(ip);
        String objectName = "casehub-" + sanitized;

        apiClient.setConfig(
                buildAddressXpath(objectName),
                "<ip-netmask>" + ip + "/32</ip-netmask>");

        apiClient.setConfig(
                buildAddressGroupMemberXpath(addressGroup),
                "<member>" + objectName + "</member>");

        String jobId = apiClient.commit();

        return new ContainmentResponse(true,
                "IP " + ip + " blocked via " + addressGroup,
                null, false,
                Map.of("paloalto_job_id", jobId,
                       "paloalto_address_object", objectName));
    }

    private ContainmentResponse executeBlockDomain(ContainmentRequest request) {
        String domain = request.parameters().get("domain").toString();
        String urlCategory = paramOrDefault(request, "urlCategory", defaultUrlCategory);

        apiClient.setConfig(
                buildUrlCategoryMemberXpath(urlCategory),
                "<member>" + domain + "</member>");

        String jobId = apiClient.commit();

        return new ContainmentResponse(true,
                "Domain " + domain + " blocked via " + urlCategory,
                null, false,
                Map.of("paloalto_job_id", jobId));
    }

    private ContainmentResponse executeNetworkSegmentation(ContainmentRequest request) {
        String sourceZone = request.parameters().get("sourceZone").toString();
        String destZone = request.parameters().get("destZone").toString();
        String ruleName = request.parameters().get("ruleName").toString();

        String element = "<from><member>" + sourceZone + "</member></from>"
                + "<to><member>" + destZone + "</member></to>"
                + "<source><member>any</member></source>"
                + "<destination><member>any</member></destination>"
                + "<application><member>any</member></application>"
                + "<service><member>application-default</member></service>"
                + "<action>deny</action>"
                + "<log-end>yes</log-end>";

        apiClient.setConfig(buildSecurityRuleXpath(ruleName), element);

        String jobId = apiClient.commit();

        return new ContainmentResponse(true,
                "Zone segmentation " + sourceZone + "→" + destZone
                        + " deny rule " + ruleName + " applied",
                null, false,
                Map.of("paloalto_job_id", jobId,
                       "paloalto_rule_name", ruleName));
    }

    boolean isSupportedAction(String actionType) {
        return SUPPORTED_ACTIONS.contains(actionType);
    }

    ContainmentResponse validateAndExtract(ContainmentRequest request) {
        if (!isSupportedAction(request.actionType())) {
            return new ContainmentResponse(false, null,
                    "Unsupported PAN-OS action type: " + request.actionType(),
                    false, Map.of());
        }

        return switch (request.actionType()) {
            case "block.ip" -> requireParam(request, "ip");
            case "block.domain" -> requireParam(request, "domain");
            case "network.segmentation" -> {
                ContainmentResponse r = requireParam(request, "sourceZone");
                if (r != null) yield r;
                r = requireParam(request, "destZone");
                if (r != null) yield r;
                yield requireParam(request, "ruleName");
            }
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

    private String paramOrDefault(ContainmentRequest request, String param, String defaultValue) {
        Object value = request.parameters().get(param);
        return (value != null && !value.toString().isBlank()) ? value.toString() : defaultValue;
    }

    static String sanitizeIp(String ip) {
        return ip.replace('.', '-');
    }

    String buildAddressXpath(String objectName) {
        return "/config/devices/entry[@name='" + deviceName + "']"
                + "/vsys/entry[@name='" + vsys + "']"
                + "/address/entry[@name='" + objectName + "']";
    }

    String buildAddressGroupMemberXpath(String groupName) {
        return "/config/devices/entry[@name='" + deviceName + "']"
                + "/vsys/entry[@name='" + vsys + "']"
                + "/address-group/entry[@name='" + groupName + "']/static";
    }

    String buildUrlCategoryMemberXpath(String categoryName) {
        return "/config/devices/entry[@name='" + deviceName + "']"
                + "/vsys/entry[@name='" + vsys + "']"
                + "/profiles/custom-url-category/entry[@name='" + categoryName + "']/list";
    }

    String buildSecurityRuleXpath(String ruleName) {
        return "/config/devices/entry[@name='" + deviceName + "']"
                + "/vsys/entry[@name='" + vsys + "']"
                + "/rulebase/security/rules/entry[@name='" + ruleName + "']";
    }
}
```

- [ ] **Step 4: Implement PaloAltoHealthCheck**

```java
package io.casehub.soc.connector.paloalto;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;

import java.util.Map;

@Path("/paloalto/health")
@ApplicationScoped
public class PaloAltoHealthCheck {

    @Inject
    PaloAltoApiClient apiClient;

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response health() {
        try {
            apiClient.systemInfo();
            return Response.ok(Map.of("status", "UP")).build();
        } catch (Exception e) {
            return Response.status(503)
                    .entity(Map.of("status", "DOWN", "reason", e.getMessage()))
                    .build();
        }
    }
}
```

- [ ] **Step 5: Implement PaloAltoConfig**

```java
package io.casehub.soc.connector.paloalto;

import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import jakarta.inject.Singleton;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
public class PaloAltoConfig {

    @Produces
    @Singleton
    public PaloAltoApiClient paloAltoApiClient(
            @ConfigProperty(name = "casehub.soc.paloalto.api-base",
                             defaultValue = "https://firewall.corp.local") String apiBase,
            @ConfigProperty(name = "casehub.soc.paloalto.api-key",
                             defaultValue = "") String apiKey,
            @ConfigProperty(name = "casehub.soc.paloalto.vsys",
                             defaultValue = "vsys1") String vsys,
            @ConfigProperty(name = "casehub.soc.paloalto.device-name",
                             defaultValue = "localhost.localdomain") String deviceName,
            @ConfigProperty(name = "casehub.soc.paloalto.commit-poll-interval-ms",
                             defaultValue = "2000") long commitPollIntervalMs,
            @ConfigProperty(name = "casehub.soc.paloalto.commit-timeout-ms",
                             defaultValue = "60000") long commitTimeoutMs) {
        if (apiKey.isBlank()) {
            return PaloAltoApiClient.unconfigured();
        }
        return new PaloAltoApiClient(apiBase, apiKey, vsys, deviceName,
                commitPollIntervalMs, commitTimeoutMs);
    }
}
```

- [ ] **Step 6: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=PaloAltoContainmentConnectorTest,PaloAltoApiClientTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/soc/connector/paloalto/ app/src/test/java/io/casehub/soc/connector/paloalto/
git commit -m "feat(#53): add PaloAltoContainmentConnector, health check, and CDI config

Refs #53"
```

---

## Batch 2: WireMock contract tests and config

### Task 3: WireMock contract tests + application.properties config

**Files:**
- Create: `app/src/test/java/io/casehub/soc/connector/paloalto/PaloAltoContainmentWireMockTest.java`
- Modify: `app/src/main/resources/application.properties` (add Palo Alto config defaults)

**Interfaces:**
- Consumes: `PaloAltoContainmentConnector` (Task 2), `PaloAltoApiClient` (Task 1)
- Produces: Contract test coverage, Palo Alto config properties

- [ ] **Step 1: Write WireMock contract test**

```java
package io.casehub.soc.connector.paloalto;

import com.github.tomakehurst.wiremock.WireMockServer;
import com.github.tomakehurst.wiremock.client.WireMock;
import io.casehub.soc.engine.spi.ContainmentRequest;
import io.casehub.soc.engine.spi.ContainmentResponse;
import org.junit.jupiter.api.AfterAll;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static com.github.tomakehurst.wiremock.client.WireMock.*;
import static org.assertj.core.api.Assertions.assertThat;

class PaloAltoContainmentWireMockTest {

    static WireMockServer wireMock;
    PaloAltoContainmentConnector connector;
    PaloAltoApiClient apiClient;

    private static final String CONFIG_SUCCESS =
            "<response status=\"success\" code=\"20\"><msg>command succeeded</msg></response>";
    private static final String COMMIT_SUCCESS =
            "<response status=\"success\" code=\"19\"><result><msg>"
            + "<line>Commit job enqueued with jobid 42</line></msg>"
            + "<job>42</job></result></response>";
    private static final String JOB_COMPLETE =
            "<response status=\"success\"><result><job><id>42</id>"
            + "<status>FIN</status><result>OK</result>"
            + "<progress>100</progress></job></result></response>";
    private static final String JOB_FAILED =
            "<response status=\"success\"><result><job><id>42</id>"
            + "<status>FIN</status><result>FAIL</result>"
            + "<details><line>Configuration commit failed</line></details>"
            + "<progress>100</progress></job></result></response>";
    private static final String CONFIG_ERROR =
            "<response status=\"error\" code=\"12\">"
            + "<msg><line>Object not found</line></msg></response>";

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

    @BeforeEach
    void setUp() {
        wireMock.resetAll();

        String baseUrl = "http://localhost:" + wireMock.port();
        apiClient = new PaloAltoApiClient(baseUrl, "test-api-key", "vsys1",
                "localhost.localdomain", 100, 5000);

        connector = new PaloAltoContainmentConnector();
        connector.apiClient = apiClient;
        connector.deviceName = "localhost.localdomain";
        connector.vsys = "vsys1";
        connector.defaultAddressGroup = "casehub-blocked-ips";
        connector.defaultUrlCategory = "casehub-blocked-domains";
    }

    private void stubConfigSuccess() {
        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("config"))
                .willReturn(okXml(CONFIG_SUCCESS)));
    }

    private void stubCommitAndJobSuccess() {
        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("commit"))
                .willReturn(okXml(COMMIT_SUCCESS)));

        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("op"))
                .willReturn(okXml(JOB_COMPLETE)));
    }

    @Test
    void successfulIpBlock() {
        stubConfigSuccess();
        stubCommitAndJobSuccess();

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.ip", Map.of("ip", "10.0.0.5"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isTrue();
        assertThat(response.details()).contains("10.0.0.5");
        assertThat(response.details()).contains("casehub-blocked-ips");
        assertThat(response.metadata()).containsEntry("paloalto_job_id", "42");
        assertThat(response.metadata()).containsEntry("paloalto_address_object", "casehub-10-0-0-5");

        verify(2, postRequestedFor(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("config"))
                .withQueryParam("key", equalTo("test-api-key")));
    }

    @Test
    void successfulDomainBlock() {
        stubConfigSuccess();
        stubCommitAndJobSuccess();

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.domain", Map.of("domain", "evil.example.com"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isTrue();
        assertThat(response.details()).contains("evil.example.com");
        assertThat(response.details()).contains("casehub-blocked-domains");
        assertThat(response.metadata()).containsEntry("paloalto_job_id", "42");
    }

    @Test
    void successfulNetworkSegmentation() {
        stubConfigSuccess();
        stubCommitAndJobSuccess();

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "network.segmentation",
                Map.of("sourceZone", "trust", "destZone", "dmz", "ruleName", "casehub-seg-001"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isTrue();
        assertThat(response.details()).contains("trust");
        assertThat(response.details()).contains("dmz");
        assertThat(response.metadata()).containsEntry("paloalto_job_id", "42");
        assertThat(response.metadata()).containsEntry("paloalto_rule_name", "casehub-seg-001");
    }

    @Test
    void configErrorReturnsFailure() {
        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("config"))
                .willReturn(okXml(CONFIG_ERROR)));

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.ip", Map.of("ip", "10.0.0.5"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("Object not found");
        assertThat(response.retryable()).isFalse();
    }

    @Test
    void commitFailureReturnsRetryableError() {
        stubConfigSuccess();

        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("commit"))
                .willReturn(okXml(COMMIT_SUCCESS)));

        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("op"))
                .willReturn(okXml(JOB_FAILED)));

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.ip", Map.of("ip", "10.0.0.5"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("commit failed");
        assertThat(response.retryable()).isTrue();
    }

    @Test
    void authenticationFailureReturnsNonRetryableError() {
        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("config"))
                .willReturn(aResponse().withStatus(403)));

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.ip", Map.of("ip", "10.0.0.5"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("authentication failed");
        assertThat(response.retryable()).isFalse();
    }

    @Test
    void serverErrorReturnsRetryableError() {
        stubFor(post(urlPathEqualTo("/api/"))
                .withQueryParam("type", equalTo("config"))
                .willReturn(aResponse().withStatus(500)));

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.ip", Map.of("ip", "10.0.0.5"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isFalse();
        assertThat(response.retryable()).isTrue();
    }

    @Test
    void missingIpParameterReturnsFailure() {
        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.ip", Map.of(),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isFalse();
        assertThat(response.errorReason()).contains("ip");
        assertThat(response.retryable()).isFalse();

        wireMock.verify(0, postRequestedFor(urlPathEqualTo("/api/")));
    }

    @Test
    void customAddressGroupOverridesDefault() {
        stubConfigSuccess();
        stubCommitAndJobSuccess();

        ContainmentResponse response = connector.execute(new ContainmentRequest(
                "block.ip", Map.of("ip", "10.0.0.5", "addressGroup", "custom-group"),
                "case-1", "INC-001", "analyst@corp.com", "tenant-1", 90_000L));

        assertThat(response.success()).isTrue();
        assertThat(response.details()).contains("custom-group");
    }
}
```

- [ ] **Step 2: Run WireMock tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=PaloAltoContainmentWireMockTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: All tests PASS

- [ ] **Step 3: Add Palo Alto config defaults to application.properties**

Append to `app/src/main/resources/application.properties`:

```properties
# Palo Alto PAN-OS connector (credentials from env vars in production)
casehub.soc.paloalto.api-base=https://firewall.corp.local
casehub.soc.paloalto.api-key=${PALOALTO_API_KEY:}
casehub.soc.paloalto.vsys=vsys1
casehub.soc.paloalto.device-name=localhost.localdomain
casehub.soc.paloalto.default-address-group=casehub-blocked-ips
casehub.soc.paloalto.default-url-category=casehub-blocked-domains
casehub.soc.paloalto.commit-poll-interval-ms=2000
casehub.soc.paloalto.commit-timeout-ms=60000
```

- [ ] **Step 4: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 5: Commit**

```bash
git add app/src/test/java/io/casehub/soc/connector/paloalto/ app/src/main/resources/application.properties
git commit -m "test(#53): add Palo Alto WireMock contract tests and config defaults

Refs #53"
```

---

## References

- `specs/issue-34-fix-upstream-api-breaks/2026-09-14-palo-alto-containment-connector-design.md` — design spec
- `specs/issue-50-connector-containment-runtimes/2026-09-14-connector-containment-runtimes-design.md` — parent spec (Batch 3)
- `CrowdStrikeContainmentConnector.java:28` — reference connector (same pattern)
- `CrowdStrikeOAuth2Client.java:13` — reference auth client
- `CrowdStrikeConfig.java:9` — reference CDI producer
- `CrowdStrikeContainmentWireMockTest.java:17` — reference WireMock test
- `CrowdStrikeContainmentConnectorTest.java:11` — reference unit test
- `ContainmentRequest.java:5` — API contract (request)
- `ContainmentResponse.java:6` — API contract (response)
- `HttpContainmentExecutor.java:25` — executor that routes to connectors
- GitHub #53 — focal issue
- GitHub #50 — parent epic
