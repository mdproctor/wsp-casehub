# Palo Alto PAN-OS Containment Connector — Design Spec

**Issue:** #53 — Palo Alto containment connector — IP/domain blocking and network segmentation
**Parent:** Epic #50 — Connector-based containment runtimes (Batch 3)
**Date:** 2026-09-14

---

## Problem

`HttpContainmentExecutor` routes containment actions to HTTP connector endpoints, but only CrowdStrike Falcon (`isolate.host`, `wipe.endpoint`) and the simulated connector exist. Three action types — `block.ip`, `block.domain`, `network.segmentation` — are routed to the simulated connector in dev/test. Real containment for network-layer threats requires a Palo Alto PAN-OS integration.

## Approach

In-process JAX-RS endpoint within SOC (D1), following the CrowdStrike connector pattern. Receives `ContainmentRequest`, translates to PAN-OS XML API calls, commits the configuration change, and returns `ContainmentResponse`.

Key differences from CrowdStrike:
- **XML API** — PAN-OS uses an XML-based REST API, not JSON
- **API key auth** — long-lived key passed as query parameter (no token refresh lifecycle)
- **Two-phase commit** — changes are staged in candidate config, then committed; commit is async with a job ID that must be polled to completion

## PAN-OS XML API

### Authentication

All API calls include the API key as a query parameter:

```
POST https://<firewall>/api/?key=<apikey>&type=<type>&...
```

The API key is generated once from the firewall admin UI or via `POST /api/?type=keygen&user=<user>&password=<pass>`. It does not expire unless revoked. No refresh lifecycle needed.

### Config Operations (type=config)

All containment actions stage changes in the candidate configuration:

```
POST /api/?key=<key>&type=config&action=set&xpath=<xpath>&element=<xml>
```

Response (success):
```xml
<response status="success" code="20"><msg>command succeeded</msg></response>
```

Response (error):
```xml
<response status="error" code="12"><msg><line>Object not found</line></msg></response>
```

### block.ip — Create Address Object + Add to Group

**Step 1: Create address object**

```
xpath=/config/devices/entry[@name='<device>']/vsys/entry[@name='<vsys>']/address/entry[@name='casehub-<sanitized-ip>']
element=<ip-netmask><ip>/32</ip-netmask>
```

Where `<sanitized-ip>` replaces dots with dashes (e.g., `192.168.1.100` → `casehub-192-168-1-100`).

**Step 2: Add to address group**

```
xpath=/config/devices/entry[@name='<device>']/vsys/entry[@name='<vsys>']/address-group/entry[@name='<group>']/static
element=<member>casehub-<sanitized-ip></member>
```

The address group (default: `casehub-blocked-ips`) must already exist and be referenced by a deny rule. The connector adds members; it does not create the group or its referencing rule.

**Required parameters:** `ip` — the IP address to block.
**Optional parameters:** `addressGroup` — override default group name.

### block.domain — Add to Custom URL Category

```
xpath=/config/devices/entry[@name='<device>']/vsys/entry[@name='<vsys>']/profiles/custom-url-category/entry[@name='<category>']/list
element=<member><domain></member>
```

The custom URL category (default: `casehub-blocked-domains`) must already exist and be referenced by a URL filtering profile attached to a security rule.

**Required parameters:** `domain` — the domain to block.
**Optional parameters:** `urlCategory` — override default category name.

### network.segmentation — Create Inter-Zone Deny Rule

```
xpath=/config/devices/entry[@name='<device>']/vsys/entry[@name='<vsys>']/rulebase/security/rules/entry[@name='<ruleName>']
element=<from><member><sourceZone></member></from><to><member><destZone></member></to><source><member>any</member></source><destination><member>any</member></destination><application><member>any</member></application><service><member>application-default</member></service><action>deny</action><log-end>yes</log-end>
```

Creates a new deny rule between zones. Unlike `block.ip` and `block.domain`, this creates a standalone rule rather than adding to a pre-configured group.

**Required parameters:** `sourceZone`, `destZone`, `ruleName`.

### Commit (type=commit)

```
POST /api/?key=<key>&type=commit&cmd=<commit></commit>
```

Response:
```xml
<response status="success" code="19"><result><msg><line>Commit job enqueued with jobid 42</line></msg><job>42</job></result></response>
```

### Commit Job Poll (type=op)

```
POST /api/?key=<key>&type=op&cmd=<show><jobs><id>42</id></jobs></show>
```

Response (in progress):
```xml
<response status="success"><result><job><id>42</id><status>ACT</status><progress>50</progress></job></result></response>
```

Response (complete):
```xml
<response status="success"><result><job><id>42</id><status>FIN</status><result>OK</result><progress>100</progress></job></result></response>
```

Response (failed):
```xml
<response status="success"><result><job><id>42</id><status>FIN</status><result>FAIL</result><details><line>Configuration commit failed</line></details></job></result></response>
```

Poll interval: 2 seconds (configurable). Timeout: 60 seconds (configurable).

## New Components

### PaloAltoContainmentConnector

JAX-RS endpoint at `/paloalto/containment` in `app/`.

```java
@Path("/paloalto/containment")
@ApplicationScoped
public class PaloAltoContainmentConnector {

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public ContainmentResponse execute(ContainmentRequest request) {
        // 1. Validate action type and required parameters
        // 2. Execute PAN-OS config operations for the action type
        // 3. Commit candidate config
        // 4. Poll commit job to completion
        // 5. Return ContainmentResponse with job ID and status
    }
}
```

**Action mapping:**

| ContainmentRequest.actionType | PAN-OS operations | Required parameters |
|---|---|---|
| `block.ip` | Create address object → add to address group → commit | `ip` |
| `block.domain` | Add domain to custom URL category → commit | `domain` |
| `network.segmentation` | Create inter-zone deny rule → commit | `sourceZone`, `destZone`, `ruleName` |

**Module:** `app/`
**Package:** `io.casehub.soc.connector.paloalto`

### PaloAltoApiClient

Handles PAN-OS XML API calls. Plain class (not CDI-annotated), produced by `PaloAltoConfig`.

```java
public class PaloAltoApiClient {

    public PaloAltoApiResponse setConfig(String xpath, String element) {
        // POST /api/?key=<key>&type=config&action=set&xpath=<xpath>&element=<element>
        // Parse XML response, return status + message
    }

    public String commit() {
        // POST /api/?key=<key>&type=commit&cmd=<commit></commit>
        // Parse job ID from response
        // Poll until FIN, return job result
    }

    public PaloAltoApiResponse systemInfo() {
        // POST /api/?key=<key>&type=op&cmd=<show><system><info></info></system></show>
        // Used by health check
    }
}
```

API key is stored as a field — no refresh lifecycle needed. Thread-safe: no mutable state (API key is immutable after construction).

**Config:**
```properties
casehub.soc.paloalto.api-base=https://firewall.corp.local
casehub.soc.paloalto.api-key=${PALOALTO_API_KEY:}
casehub.soc.paloalto.vsys=vsys1
casehub.soc.paloalto.device-name=localhost.localdomain
casehub.soc.paloalto.default-address-group=casehub-blocked-ips
casehub.soc.paloalto.default-url-category=casehub-blocked-domains
casehub.soc.paloalto.commit-poll-interval-ms=2000
casehub.soc.paloalto.commit-timeout-ms=60000
```

**Package:** `io.casehub.soc.connector.paloalto`

### PaloAltoHealthCheck

```java
@Path("/paloalto/health")
@ApplicationScoped
public class PaloAltoHealthCheck {

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public Response health() {
        // Call systemInfo() — verifies connectivity and API key validity
        // 200 with {"status": "UP", "hostname": "..."} if successful
        // 503 with {"status": "DOWN", "reason": "..."} if not
    }
}
```

**Package:** `io.casehub.soc.connector.paloalto`

### PaloAltoConfig

CDI producer for `PaloAltoApiClient`.

```java
@ApplicationScoped
public class PaloAltoConfig {

    @Produces
    @Singleton
    public PaloAltoApiClient paloAltoApiClient(...) {
        if (apiKey.isBlank()) {
            return PaloAltoApiClient.unconfigured();
        }
        return new PaloAltoApiClient(apiBase, apiKey, vsys, deviceName,
                commitPollIntervalMs, commitTimeoutMs);
    }
}
```

**Package:** `io.casehub.soc.connector.paloalto`

### PaloAltoApiException

Runtime exception for PAN-OS API errors (analogous to `CrowdStrikeAuthException`).

```java
public class PaloAltoApiException extends RuntimeException {
    private final int code;
    // ...
}
```

### PaloAltoApiResponse

Simple record for parsed XML API responses.

```java
public record PaloAltoApiResponse(String status, int code, String message) {
    public boolean isSuccess() { return "success".equals(status); }
}
```

## Configuration

### application.properties additions

```properties
# Palo Alto connector endpoint (for HttpContainmentExecutor routing)
casehub.soc.containment.endpoints.paloalto.url=http://localhost:${quarkus.http.port}/paloalto/containment
casehub.soc.containment.endpoints.paloalto.method=POST
casehub.soc.containment.endpoints.paloalto.timeout-seconds=90

# Palo Alto PAN-OS API credentials (env vars in production)
casehub.soc.paloalto.api-base=https://firewall.corp.local
casehub.soc.paloalto.api-key=${PALOALTO_API_KEY:}
casehub.soc.paloalto.vsys=vsys1
casehub.soc.paloalto.device-name=localhost.localdomain
casehub.soc.paloalto.default-address-group=casehub-blocked-ips
casehub.soc.paloalto.default-url-category=casehub-blocked-domains
casehub.soc.paloalto.commit-poll-interval-ms=2000
casehub.soc.paloalto.commit-timeout-ms=60000
```

The endpoint timeout (90s) is higher than CrowdStrike (30s) to account for commit polling. It must exceed `commit-timeout-ms` (60s) — otherwise `HttpContainmentExecutor` times out the HTTP call before the commit completes, producing a false "unreachable" error while the firewall commit proceeds unsupervised.

In dev/test, routing stays pointed at `sim`. In production, override `casehub.soc.containment.routing.block-ip=paloalto`, `casehub.soc.containment.routing.block-domain=paloalto`, `casehub.soc.containment.routing.network-segmentation=paloalto`.

## Testing Strategy

### Unit tests

- **PaloAltoContainmentConnectorTest** — action mapping, parameter validation (missing ip, missing domain, missing zone params), response mapping for success/error
- **PaloAltoApiClientTest** — XML template generation, API response parsing, commit job polling logic, unconfigured state handling

### Contract tests (WireMock)

- **PaloAltoContainmentWireMockTest** — full round-trip:
  - Stub PAN-OS config set endpoint → return success/error XML
  - Stub commit endpoint → return job ID
  - Stub job poll endpoint → return in-progress then complete
  - POST `ContainmentRequest` to `/paloalto/containment`
  - Verify correct PAN-OS API calls were made (API key, xpath, element)
  - Verify `ContainmentResponse` mapping

**WireMock scenarios:**

1. Successful IP block — config set (address + group) → commit → poll FIN/OK → success
2. Successful domain block — config set (URL category) → commit → poll FIN/OK → success
3. Successful network segmentation — config set (deny rule) → commit → poll FIN/OK → success
4. PAN-OS config error — address object already exists → failure (retryable=false)
5. PAN-OS commit fails — job finishes with FAIL → failure (retryable=true)
6. PAN-OS returns 403 — invalid API key → failure (retryable=false)
7. PAN-OS unreachable — connection refused → failure (retryable=true)
8. Commit timeout — poll exceeds timeout → failure (retryable=true)
9. Missing required parameter (ip/domain/zones) → failure (retryable=false)

## Failure Modes

| Failure | Connector behavior |
|---|---|
| Missing required parameter | `ContainmentResponse(false, null, "Missing required parameter: <param>", false, {})` |
| PAN-OS API key invalid (403) | `ContainmentResponse(false, null, "PAN-OS authentication failed", false, {})` |
| Config set fails (error XML) | `ContainmentResponse(false, null, "PAN-OS config error: <message>", false, {"paloalto_code": N})` |
| Commit job fails (FIN/FAIL) | `ContainmentResponse(false, null, "PAN-OS commit failed: <details>", true, {"paloalto_job_id": "N"})` |
| Commit timeout | `ContainmentResponse(false, null, "PAN-OS commit timed out after Ns", true, {"paloalto_job_id": "N"})` |
| PAN-OS unreachable | `ContainmentResponse(false, null, "PAN-OS unreachable: <reason>", true, {})` |
| PAN-OS 429 rate limited | `ContainmentResponse(false, null, "PAN-OS rate limited", true, {})` |
| PAN-OS 5xx | `ContainmentResponse(false, null, "PAN-OS server error: <status>", true, {})` |
| Address group/URL category not found | `ContainmentResponse(false, null, "PAN-OS config error: <group> not found — pre-configure the address group", false, {})` |
| Success (block.ip) | `ContainmentResponse(true, "IP <ip> blocked via <group>", null, false, {"paloalto_job_id": "N", "paloalto_address_object": "casehub-<ip>"})` |
| Success (block.domain) | `ContainmentResponse(true, "Domain <domain> blocked via <category>", null, false, {"paloalto_job_id": "N"})` |
| Success (network.segmentation) | `ContainmentResponse(true, "Zone segmentation <src>→<dst> deny rule <name> applied", null, false, {"paloalto_job_id": "N", "paloalto_rule_name": "<name>"})` |

## Prerequisites

The firewall must be pre-configured with:
1. An address group named `casehub-blocked-ips` (or as configured) referenced by a deny security rule
2. A custom URL category named `casehub-blocked-domains` (or as configured) referenced by a URL filtering profile attached to a security rule
3. An API key with sufficient privileges for config set + commit operations

The connector validates that these groups exist on first use and returns clear error messages if they are missing.

## References

- `CrowdStrikeContainmentConnector.java:28` — reference connector implementation (same pattern)
- `CrowdStrikeOAuth2Client.java:13` — reference auth client (PaloAltoApiClient is simpler — no token lifecycle)
- `CrowdStrikeConfig.java:9` — reference CDI producer
- `HttpContainmentExecutor.java:25` — executor that routes to this connector
- `ContainmentRequest.java:5` / `ContainmentResponse.java:6` — API contract
- `ContainmentEndpointResolver.java` — config-driven endpoint resolution
- `specs/issue-50-connector-containment-runtimes/2026-09-14-connector-containment-runtimes-design.md` — parent spec (Batch 3)
- PAN-OS XML API: `POST /api/?type=config` — configuration operations
- PAN-OS XML API: `POST /api/?type=commit` — commit candidate config
- PAN-OS XML API: `POST /api/?type=op` — operational commands (job status, system info)
