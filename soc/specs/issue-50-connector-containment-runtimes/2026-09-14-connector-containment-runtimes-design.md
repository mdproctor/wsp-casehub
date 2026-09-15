# Connector-Based Containment Runtimes — Design Spec

**Epic:** #50 — Connector-based containment runtimes — HTTP/MCP workers for EDR and firewall APIs
**Date:** 2026-09-14
**Revision:** 2 — incorporates post-spec review findings (structure, coherence, robustness)

---

## Problem

All containment actions in casehub-soc execute through `LoggingContainmentExecutor` — a `@DefaultBean` stub that logs the action and returns success. No actual external system integration exists. Real containment requires calling external APIs: CrowdStrike Falcon for host isolation/wipe, Palo Alto for network segmentation and IP/domain blocking, Okta/Azure AD for credential revocation and account disabling.

Without real connectors:
- Containment actions are theatre — the audit trail records "executed" but nothing happened
- DORA response timelines measure log-write latency, not actual containment latency
- The `ContainmentExecutor` SPI (delivered in #40) is validated only against the no-op stub
- SOC cannot demonstrate end-to-end incident response to practitioners evaluating CaseHub

## Approach

Build `HttpContainmentExecutor` — a new `@ApplicationScoped` implementation of `ContainmentExecutor` that routes action types to external HTTP connectors. Each external integration (CrowdStrike, Palo Alto, Okta) is a lightweight sidecar HTTP service implementing a standard containment API contract. SOC configures which action types route to which connector via Quarkus config.

### Key architectural choice

SOC owns its own endpoint resolver (`ContainmentEndpointResolver`) — a simple `@ApplicationScoped` bean that reads `casehub.soc.containment.endpoints.*` config at `@PostConstruct`. No dependency on the workers repo.

**Why not use workers-http:** `HttpEndpointResolver.initialize()` is package-private and called by `HttpWorkerRuntime` during the worker runtime lifecycle — not at CDI `@PostConstruct`. Injecting it from SOC gives an empty endpoint map. The 3-tier resolution (SPI routes → config → EndpointRegistry) is more infrastructure than SOC needs. SOC's containment endpoints are static config — a simple config reader is sufficient and avoids the lifecycle coupling.

### What stays unchanged

- `ContainmentExecutor` SPI — synchronous interface, no changes
- `LoggingContainmentExecutor` — `@DefaultBean` fallback when no connectors configured
- Case plan YAML — `containment-execution` binding fires the same way
- Ledger observers — `SocContainmentLedgerObserver` writes the same audit entries

## Pipeline Flow

```
RuleContainmentExecutionWorker (local, minor change — see below)
    │
    ├── gate check: actionGateApproved / actionGateRejected / autonomous
    ├── skip path: no PlannedAction or rejected → WorkerResult(executed=false)
    │
    └── execute path:
            │
            HttpContainmentExecutor (@ApplicationScoped, new)
                │
                ├── action type → endpoint tag (config)
                │     casehub.soc.containment.routing.isolate-host=crowdstrike
                │
                ├── endpoint tag → resolved URL (ContainmentEndpointResolver)
                │     casehub.soc.containment.endpoints.crowdstrike.url=http://...
                │
                ├── HTTP POST to connector
                │     POST /containment  { actionType, parameters, context }
                │
                └── parse response → ContainmentResult
                      success → ContainmentResult.success(details, timestamp)
                      failure → ContainmentResult.failure(errorReason, retryable)
```

## New Components

### HttpContainmentExecutor

`@ApplicationScoped` implementation of `ContainmentExecutor` in `app/`. Displaces `LoggingContainmentExecutor` (`@DefaultBean`) automatically — any non-default bean displaces a `@DefaultBean` without needing `@Alternative` or `@Priority`.

```java
@ApplicationScoped
public class HttpContainmentExecutor implements ContainmentExecutor {

    @Inject ContainmentEndpointResolver endpointResolver;
    @Inject Vertx vertx;
    @Inject ObjectMapper objectMapper;

    Map<String, String> actionRouting; // action type → endpoint tag

    @PostConstruct
    void init() {
        // Read casehub.soc.containment.routing.* from MicroProfile Config
        // Keys use hyphens (isolate-host), converted to dots (isolate.host) for lookup
    }

    @Override
    public ContainmentResult execute(String actionType, Map<String, Object> parameters,
                                     ContainmentContext context) {
        String endpointTag = actionRouting.get(actionType);
        if (endpointTag == null) {
            LOG.infof("No routing for %s — logging only", actionType);
            return ContainmentResult.success("Logged (no connector): " + actionType, Instant.now());
        }

        ContainmentEndpoint endpoint = endpointResolver.resolve(endpointTag);
        // POST ContainmentRequest to endpoint.url(), parse ContainmentResponse
    }
}
```

**Config loading:** Reads `casehub.soc.containment.routing.*` at `@PostConstruct`. Config keys use hyphens (`isolate-host`) to avoid MicroProfile Config hierarchy interpretation of dots. Converted to dot format (`isolate.host`) when building the routing map, matching `SocActionType.actionType()` format.

**HTTP call pattern:** Synchronous Vert.x WebClient POST:
- 2xx → parse `ContainmentResponse` body → `ContainmentResult.success()`
- 2xx with unparseable body → `ContainmentResult.failure(retryable=false)` — connector returned invalid JSON
- 429 → `ContainmentResult.failure(retryable=true)`
- 4xx → `ContainmentResult.failure(retryable=false)` — permanent fault
- 5xx → `ContainmentResult.failure(retryable=true)` — transient fault
- Connection timeout/refused/DNS failure → `ContainmentResult.failure(retryable=true)`

**Timeout:** Uses `ContainmentEndpoint.timeoutSeconds()` from config. Falls back to `ContainmentContext.timeoutMs()` if not set.

**Unmapped action types:** When `actionRouting` has no entry for an action type, `HttpContainmentExecutor` logs the action at INFO level and returns `ContainmentResult.success()` — same behavior as `LoggingContainmentExecutor`. This allows incremental rollout: configure routing for the connectors you have, remaining actions log as before.

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

### ContainmentEndpointResolver

Simple `@ApplicationScoped` config reader in `app/`. Reads `casehub.soc.containment.endpoints.*` properties at `@PostConstruct`.

```java
@ApplicationScoped
public class ContainmentEndpointResolver {

    Map<String, ContainmentEndpoint> endpoints; // tag → endpoint

    @PostConstruct
    void init() {
        // Read casehub.soc.containment.endpoints.<tag>.url, .method, .timeout-seconds
    }

    ContainmentEndpoint resolve(String endpointTag) {
        ContainmentEndpoint ep = endpoints.get(endpointTag);
        if (ep == null) throw new IllegalStateException("No endpoint for tag: " + endpointTag);
        return ep;
    }
}
```

**`ContainmentEndpoint` record:** `url` (String), `method` (String, default POST), `timeoutSeconds` (int, default 30).

No dependency on workers-http. Reads config with the same pattern as `HttpEndpointResolver.loadConfigEndpoints()` but is self-contained in SOC.

**Module:** `app/`
**Package:** `io.casehub.soc.engine`

### ContainmentRequest (API contract — request)

Record in `api/` defining the JSON body POSTed to sidecar connectors.

```java
public record ContainmentRequest(
    String actionType,
    Map<String, Object> parameters,
    String caseId,
    String incidentId,
    String approver,
    String tenancyId,
    long timeoutMs
) {}
```

Constructed from `ContainmentExecutor.execute()` arguments. The connector receives everything it needs to call the external API and return a result.

**PII trust boundary:** The `approver` field contains a user email (e.g., `analyst-jane@corp.com`). Sidecar connectors are co-deployed within the same trust boundary (same cluster, same namespace) — they are internal infrastructure, not external services. The approver identity is needed by some APIs (e.g., audit logging on the CrowdStrike side). If connectors are ever exposed across trust boundaries, `approver` should be hashed or omitted — but that is not the current design.

**Module:** `api/`
**Package:** `io.casehub.soc.engine.spi`

### ContainmentResponse (API contract — response)

Record in `api/` defining the JSON body returned by sidecar connectors.

```java
public record ContainmentResponse(
    boolean success,
    String details,
    String errorReason,
    boolean retryable,
    Map<String, Object> metadata
) {}
```

Maps directly to `ContainmentResult`. The `metadata` field allows connectors to return integration-specific data (e.g., CrowdStrike session ID, Palo Alto rule ID) without changing the contract.

**Module:** `api/`
**Package:** `io.casehub.soc.engine.spi`

### SimulatedContainmentConnector

Quarkus JAX-RS endpoint in `app/` providing a simulated connector for dev/test. Implements the same API contract the real connectors will use.

```java
@Path("/sim/containment")
@ApplicationScoped
public class SimulatedContainmentConnector {

    @POST
    @Consumes(MediaType.APPLICATION_JSON)
    @Produces(MediaType.APPLICATION_JSON)
    public ContainmentResponse execute(ContainmentRequest request) {
        // Configurable: delay, failure rate, response shape per action type
    }
}
```

**Behavior:**
- Configurable delay per action type (`casehub.soc.sim.delay-ms.isolate-host=2000`)
- Configurable failure rate (`casehub.soc.sim.failure-rate.isolate-host=0.1` → 10% failures)
- Failed responses return `retryable=true` with a realistic error message
- Response metadata includes simulated external IDs (e.g., `crowdstrike_session_id`)
- Active only in dev/test profiles (`@IfBuildProfile("dev")` or `@IfBuildProfile("test")`)

**Module:** `app/`
**Package:** `io.casehub.soc.rest`

## Configuration

### Layer 1 — Action type to endpoint tag routing

Config keys use hyphens to avoid MicroProfile Config interpreting dots as hierarchy. `HttpContainmentExecutor` converts hyphens back to dots at `@PostConstruct` when building the routing map.

```properties
# Which connector handles which action type
casehub.soc.containment.routing.isolate-host=crowdstrike
casehub.soc.containment.routing.wipe-endpoint=crowdstrike
casehub.soc.containment.routing.block-ip=paloalto
casehub.soc.containment.routing.block-domain=paloalto
casehub.soc.containment.routing.network-segmentation=paloalto
casehub.soc.containment.routing.disable-user-account=okta
casehub.soc.containment.routing.revoke-credentials=okta
casehub.soc.containment.routing.rotate-api-key=okta
# enable.enhanced.logging — no routing = logged only (no external connector)
```

### Layer 2 — Endpoint tag to URL resolution

SOC-specific config, read by `ContainmentEndpointResolver` at `@PostConstruct`. No dependency on workers-http.

```properties
# CrowdStrike Falcon connector
casehub.soc.containment.endpoints.crowdstrike.url=http://crowdstrike-connector:8080/containment
casehub.soc.containment.endpoints.crowdstrike.method=POST
casehub.soc.containment.endpoints.crowdstrike.timeout-seconds=30

# Palo Alto connector
casehub.soc.containment.endpoints.paloalto.url=http://paloalto-connector:8080/containment
casehub.soc.containment.endpoints.paloalto.method=POST
casehub.soc.containment.endpoints.paloalto.timeout-seconds=30

# Okta connector
casehub.soc.containment.endpoints.okta.url=http://okta-connector:8080/containment
casehub.soc.containment.endpoints.okta.method=POST
casehub.soc.containment.endpoints.okta.timeout-seconds=15
```

### Dev/test config (simulated connector)

```properties
# All action types route to the local simulated connector
casehub.soc.containment.routing.isolate-host=sim
casehub.soc.containment.routing.wipe-endpoint=sim
casehub.soc.containment.routing.block-ip=sim
casehub.soc.containment.routing.block-domain=sim
casehub.soc.containment.routing.network-segmentation=sim
casehub.soc.containment.routing.disable-user-account=sim
casehub.soc.containment.routing.revoke-credentials=sim
casehub.soc.containment.routing.rotate-api-key=sim

casehub.soc.containment.endpoints.sim.url=http://localhost:${quarkus.http.port}/sim/containment
casehub.soc.containment.endpoints.sim.method=POST
casehub.soc.containment.endpoints.sim.timeout-seconds=10
```

## Action Type to Integration Mapping

| Action type | SocActionType | Integration | Gate policy | Batch |
|---|---|---|---|---|
| `enable.enhanced.logging` | ENABLE_ENHANCED_LOGGING | Internal (no connector) | NEVER | — |
| `rotate.api.key` | ROTATE_API_KEY | Okta/Azure AD | CONFIDENCE_THRESHOLD | 4 |
| `block.ip` | BLOCK_IP | Palo Alto | RISK_SCORE_THRESHOLD | 3 |
| `block.domain` | BLOCK_DOMAIN | Palo Alto | RISK_SCORE_THRESHOLD | 3 |
| `disable.user.account` | DISABLE_USER_ACCOUNT | Okta/Azure AD | ALWAYS | 4 |
| `isolate.host` | ISOLATE_HOST | CrowdStrike Falcon | ALWAYS | 2 |
| `revoke.credentials` | REVOKE_CREDENTIALS | Okta/Azure AD | ALWAYS | 4 |
| `network.segmentation` | NETWORK_SEGMENTATION | Palo Alto | ALWAYS | 3 |
| `wipe.endpoint` | WIPE_ENDPOINT | CrowdStrike Falcon | ALWAYS | 2 |

## Delivery Batches

### Batch 1 — Core infrastructure + simulated connector

**Child issues to create:**
1. `HttpContainmentExecutor` + `ContainmentEndpointResolver` + `ContainmentRequest`/`ContainmentResponse` + `SimulatedContainmentConnector` — core infrastructure: executor, config resolver, API contract records, simulated dev endpoint
2. Integration test — inject alert → full pipeline → verify containment executed via simulated connector → verify ledger chain

**Definition of done:** The existing `AlertToCaseIntegrationTest` scenario executes containment through the simulated HTTP connector instead of `LoggingContainmentExecutor`. Ledger entries record the actual HTTP round-trip timing.

### Batch 2 — CrowdStrike Falcon connector

**Actions:** `isolate.host`, `wipe.endpoint`

**Connector scope:**
- Lightweight HTTP service implementing the containment API contract
- Calls CrowdStrike Falcon API: `POST /devices/entities/devices-actions/v2` for host isolation
- OAuth2 client credentials flow for authentication
- Maps `ContainmentRequest.parameters` to CrowdStrike-specific request format
- Returns CrowdStrike session ID and action status in response metadata
- Health check endpoint (`GET /health`) checking CrowdStrike API reachability

**Where it lives:** Separate project (TBD — may be in a `soc-connectors/` repo or under `casehub-connectors`).

### Batch 3 — Palo Alto connector

**Actions:** `block.ip`, `block.domain`, `network.segmentation`

**Connector scope:**
- Calls Palo Alto PAN-OS API or Panorama API
- Creates security policy rules for IP/domain blocking
- Network segmentation via zone-based policy changes
- API key authentication
- Returns rule ID and commit status in response metadata

### Batch 4 — Okta/Azure AD connector

**Actions:** `disable.user.account`, `revoke.credentials`, `rotate.api.key`

**Connector scope:**
- Calls Okta Users API or Microsoft Graph API
- User lifecycle operations (suspend, deactivate)
- Session revocation and credential reset
- OAuth2/OIDC authentication
- Returns user ID and action status in response metadata

## Existing Code Changes

### RuleContainmentExecutionWorker — minor change

The current worker calls `executor.execute()` and unconditionally maps the result to output fields. It does not distinguish retryable vs permanent failure — both produce `WorkerResult.of(output)`. For retryable failures to surface to the engine's fault pipeline, the worker needs a new code path:

When `ContainmentResult.success() == false && retryable == true`: the worker should record the failure in the output map (`executed=false`, `success=false`, `errorReason`) but still return `WorkerResult.of(output)`. The fault information propagates through the case context, and the SLA breach policy handles escalation. The engine does not retry workers automatically — retryable means "the failure is transient, the operator should investigate."

### SocInvestigationCaseDescriptor — no change

The `ContainmentExecutor` is injected via CDI. `HttpContainmentExecutor` displaces `LoggingContainmentExecutor` (`@DefaultBean`) automatically. The descriptor and worker registration are unchanged.

### application.properties — extend

Add routing config, endpoint config, and simulated connector config for dev/test profiles.

### pom.xml — no new external dependencies

No dependency on `casehub-workers-http`. All new components are self-contained in SOC. Vert.x WebClient is already available via Quarkus (transitive from `quarkus-vertx`).

## Testing Strategy

### Unit tests (batch 1)
- `HttpContainmentExecutorTest` — all code paths: successful execution, retryable failure, permanent failure, unmapped action type fallback, connection timeout, 429 retry, malformed 200 response
- `ContainmentEndpointResolverTest` — config loading, missing endpoint tag, default timeout
- `SimulatedContainmentConnectorTest` — configurable delay, failure rate, response shapes
- `ContainmentRequest`/`ContainmentResponse` — serialization round-trip

### Integration test (batch 1)
- `ConnectorContainmentIntegrationTest` — full pipeline: inject alert → triage → gate → execution via simulated HTTP connector → verify `ContainmentResult` reflects HTTP response → verify ledger chain records actual round-trip timing
- Four scenarios: gated+approved (HTTP success), autonomous (HTTP success), gated+rejected (no HTTP call), HTTP failure (retryable error, verify fault handling)

### Contract tests (batch 2+)
- Each real connector gets a contract test verifying it implements the containment API correctly
- WireMock stubs for the external API (CrowdStrike/Palo Alto/Okta) in connector tests

## Failure Modes

| Failure | HttpContainmentExecutor behavior |
|---|---|
| Connector unreachable (connection refused/timeout) | `ContainmentResult.failure(retryable=true)` |
| Connector returns 429 | `ContainmentResult.failure(retryable=true)` |
| Connector returns 4xx | `ContainmentResult.failure(retryable=false)` — permanent fault |
| Connector returns 5xx | `ContainmentResult.failure(retryable=true)` — transient fault |
| Connector returns 200 with unparseable body | `ContainmentResult.failure(retryable=false)` — connector returned invalid JSON |
| No routing configured for action type | Log at INFO, return `ContainmentResult.success()` — intentional: this action type has no external connector (e.g., `enable.enhanced.logging`) |
| `ContainmentEndpointResolver` cannot resolve endpoint tag | `ContainmentResult.failure(retryable=false)` — config error: routing points to a tag that has no endpoint URL configured |

When `ContainmentResult.retryable=true`, the `RuleContainmentExecutionWorker` returns `WorkerResult` with `executed=false`, `success=false`, and `errorReason` from the `ContainmentResult`. The failure is recorded in the case context and the SLA breach policy handles escalation. The engine does not automatically retry — "retryable" signals the failure is transient and investigation is warranted.

## References

- `ContainmentExecutor.java:5` — SPI interface (unchanged)
- `LoggingContainmentExecutor.java:15` — `@DefaultBean` fallback (unchanged)
- `RuleContainmentExecutionWorker.java:19` — local orchestrator (minor change: retryable failure path)
- `SocInvestigationCaseDescriptor.java:39` — worker registration (unchanged)
- `SocActionType.java:16` — 9 action types with gate policies
- `HttpEndpointResolver.java:25` — reference for config loading pattern (not a dependency)
- `HttpWorkerExecutionManager.java:104-125` — reference for HTTP call pattern (not a dependency)
- `SocContainmentLedgerObserver` — audit trail (unchanged)
- `ContainmentDecisionMatrix.java` — severity × tactic mapping (unchanged)
- `docs/specs/issue-40-containment-execution/2026-09-02-containment-execution-pipeline-design.md` — foundation spec
- `docs/specs/issue-40-containment-execution/decisions.md` — D2 (log-and-record default executor)
