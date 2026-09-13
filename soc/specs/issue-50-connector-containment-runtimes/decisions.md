# Decisions — Connector-Based Containment Runtimes (#50)

## D1: Full workers integration over bridge pattern

**Choice:** Wire SOC's containment execution through the workers repo WorkerRuntime/WorkerExecutionManager infrastructure. ContainmentExecutor impl delegates to external HTTP endpoints via the workers HTTP infrastructure.
**Alternatives:**
- Bridge pattern — HttpContainmentExecutor calls external APIs directly via Vert.x WebClient without using the workers repo. Simpler but reinvents retry, circuit-breaking, and fault handling.
- Thin adapter — Compile dependency on workers-http to use HttpEndpointResolver, but bypass the dispatch pipeline. Gets config resolution for free, loses retry/fault infrastructure.
**Rationale:** The workers repo already has production-grade HTTP dispatch with retry (429 handling), async callbacks, idempotency headers, URI templating, timeout management, and fault publishing. Reinventing this in SOC creates a maintenance burden and diverges from platform patterns.
**Trade-offs:** Compile dependency on workers-http module. SOC's ContainmentExecutor call path now depends on the workers infrastructure being available.
**Sources:** `HttpWorkerExecutionManager.java` (retry, fault handling), `HttpEndpointResolver.java` (3-tier endpoint resolution), `HttpWorkerRoute.java` (SPI route interface)
**Exploration:** quick
**Status:** captured

## D2: Local orchestrator with external execution

**Choice:** RuleContainmentExecutionWorker stays as the local worker handling domain logic (gate check, DORA timing, skip/execute decisions). The HttpContainmentExecutor impl handles only the external API call via workers HTTP infrastructure.
**Alternatives:**
- Push to external worker — external HTTP worker receives full case context and handles both orchestration and execution. Couples domain logic to the external service.
- Split at action dispatch — separate the decision logic and execution dispatch into two case plan bindings. More bindings, more complexity.
**Rationale:** Domain orchestration logic (gate approval checking, DORA metric computation, no-containment-path handling) is SOC-specific and should not leak to external services. The external connector only needs to know "execute this action type with these parameters" — not the full investigation context.
**Trade-offs:** Two layers of execution (local worker → ContainmentExecutor → HTTP call). Acceptable because the layers have distinct responsibilities.
**Sources:** `RuleContainmentExecutionWorker.java` (domain logic), `ContainmentExecutor.java` (SPI boundary)
**Exploration:** quick
**Status:** captured

## D3: Sidecar connector deployment model

**Choice:** Each external integration (CrowdStrike, Palo Alto, Okta) is a lightweight HTTP service deployed as a sidecar or separate container. SOC configures endpoints via `casehub.workers.http.endpoints.*` config properties.
**Alternatives:**
- In-process HTTP routes — HttpWorkerRoute CDI beans inside casehub-soc calling external APIs directly. Simpler but tightly couples external API clients to SOC deployment. Credential management leaks into SOC config.
- WireMock stubs — static response mappings. Good for contract testing but no behavior modeling.
**Rationale:** Sidecar connectors can be written in any language, deployed/scaled independently, credentialed separately. SOC stays thin — it delegates to the connector via a standard contract. This matches the workers repo philosophy where execution happens in external runtimes, not in-process.
**Trade-offs:** More operational pieces to deploy. For dev/test, a simulated connector inside SOC provides the same experience without the deployment overhead.
**Sources:** `HttpEndpointResolver.java` (config endpoint loading), `ResolvedEndpoint.java` (endpoint record)
**Exploration:** quick
**Status:** captured

## D4: Per-integration endpoint with action type in body

**Choice:** One HTTP endpoint per integration (e.g. `/crowdstrike/containment`, `/paloalto/containment`). The action type (`isolate.host`, `block.ip`) is in the JSON request body. The connector routes internally based on action type.
**Alternatives:**
- Per-action-type endpoints — one endpoint per action (e.g. `/containment/isolate-host`). Each is simpler but more endpoints to configure.
- Single universal endpoint — one `/containment` endpoint for all integrations. Simplest config but forces all action types through one connector.
**Rationale:** Per-integration granularity matches the deployment model (one sidecar per external system). Action type in the body lets the connector handle multiple related actions (CrowdStrike handles both host isolation and process quarantine). Fewer endpoints to configure than per-action-type.
**Trade-offs:** Connector must parse and route on action type internally. Acceptable — the connector knows its external system's API surface.
**Sources:** `SocActionType.java` (9 action types mapped to integrations), `HttpWorkerRoute.java` (route interface)
**Exploration:** quick
**Status:** captured

## D5: Config-driven action-to-integration routing

**Choice:** Quarkus config maps action types to endpoint tags: `casehub.soc.containment.routing.isolate-host=crowdstrike`. HttpContainmentExecutor reads the mapping and resolves the endpoint tag via the workers HTTP infrastructure.
**Alternatives:**
- Convention-based — derive endpoint tag from action type (isolate.host → containment-isolate-host). Zero config but less flexible.
- SocActionType enum carries it — add `integrationTag()` to the enum. Compile-time mapping, requires code changes to add integrations.
**Rationale:** Config-driven mapping allows per-deployment customization. Different deployments may use different EDR vendors for the same action type (CrowdStrike vs SentinelOne for host isolation). Config changes don't require code changes or recompilation.
**Trade-offs:** Requires config entries per action type. Acceptable — there are only 9 action types, and not all need to be mapped (unmapped actions fall back to LoggingContainmentExecutor).
**Sources:** `SocActionType.java` (action types), `HttpEndpointResolver.loadConfigEndpoints()` (config pattern)
**Exploration:** quick
**Status:** captured

## D6: Keep ContainmentExecutor SPI unchanged

**Choice:** ContainmentExecutor interface stays synchronous and unchanged. HttpContainmentExecutor is a new `@Alternative` impl in `app/` that displaces LoggingContainmentExecutor via standard CDI displacement. LoggingContainmentExecutor remains the `@DefaultBean` fallback.
**Alternatives:**
- Replace SPI with direct workers dispatch — remove ContainmentExecutor, have the worker dispatch via HttpEndpointResolver directly. Tighter coupling, fewer layers.
- Async SPI — change `execute()` to return `Uni<ContainmentResult>`. More honest about the execution model but a breaking change.
**Rationale:** The synchronous SPI is sufficient — the workers HTTP call is effectively synchronous from the caller's perspective (blocking Vert.x WebClient call). Keeping the SPI unchanged means LoggingContainmentExecutor and any future executors work without modification. Clean CDI displacement pattern consistent with the rest of the platform.
**Trade-offs:** Synchronous SPI means the HTTP call blocks the worker thread. Acceptable because the workers infra already handles this (HTTP timeout + fault pipeline).
**Sources:** `LoggingContainmentExecutor.java` (@DefaultBean pattern), `ContainmentExecutor.java` (SPI), `NoOpOversightGateService.java` (platform displacement pattern)
**Exploration:** quick
**Status:** captured

## D7: Fail with retryable error on connector unavailability

**Choice:** When the HTTP connector is unreachable, HttpContainmentExecutor returns `ContainmentResult.failure(retryable=true)`. The workers fault pipeline handles retry. No fallback to LoggingContainmentExecutor.
**Alternatives:**
- Fall back to logging — delegate to LoggingContainmentExecutor on HTTP failure. Pipeline completes but the action wasn't executed. Creates false sense of security.
- Configurable per action type — high-risk actions fail hard, low-risk fall back. More nuanced but more complex.
**Rationale:** Containment is safety-critical. Silently logging "containment executed" when the host wasn't actually isolated is worse than a visible failure that triggers the fault pipeline. The fault pipeline already handles retry, escalation, and human notification via SLA breach.
**Trade-offs:** Connector unavailability blocks containment until the connector recovers or the fault pipeline escalates. This is the correct behavior — the operator needs to know containment didn't happen.
**Sources:** `ContainmentResult.failure()` (retryable flag), `HttpWorkerFaultEventHandler.java` (fault pipeline), `SocSlaBreachPolicy` (escalation on timeout)
**Exploration:** quick
**Status:** captured

## D8: Incremental delivery in 4 batches

**Choice:** Deliver in 4 batches via the .plan queue: (1) core infrastructure + simulated connector, (2) CrowdStrike Falcon, (3) Palo Alto, (4) Okta/Azure AD. Each batch is reviewed before the next begins.
**Alternatives:**
- Simulated only — build the architecture, defer all real integrations. Proves the architecture but doesn't deliver the epic's definition of done.
- All at once — deliver all three real integrations in a single batch. Ambitious, harder to review incrementally.
**Rationale:** Batch 1 proves the full integration architecture end-to-end with the simulated connector. Each subsequent batch adds one real integration, allowing review of the connector pattern before it's replicated. CrowdStrike first because it has the best-documented API and covers the highest-value action (host isolation).
**Trade-offs:** 4 review cycles instead of 1. Acceptable — incremental review catches design issues before they're replicated across integrations.
**Sources:** Issue #50 definition of done (at least one real integration)
**Exploration:** quick
**Depends on:** D3 (sidecar deployment model)
**Status:** captured

## D9: Simulated connector inside casehub-soc

**Choice:** The simulated containment connector lives inside casehub-soc as a Quarkus dev-service or test resource. Models realistic EDR/firewall behavior (configurable delays, failure rates, response shapes).
**Alternatives:**
- Standalone project — separate repo modeling the real deployment shape. More realistic but more setup for batch 1.
- WireMock stubs — static response mappings in test resources. Simplest but no behavior modeling.
**Rationale:** Dev-service starts automatically during tests and dev mode. No separate project to maintain or deploy. The simulated connector implements the same API contract the real connectors will use, so it validates the contract even though it's co-located with SOC.
**Trade-offs:** Simulated connector doesn't model the deployment topology (separate container, network boundary). Acceptable for dev/test — deployment topology is validated in staging.
**Sources:** `SocDemoResource.java` (existing dev-mode pattern in SOC)
**Exploration:** quick
**Depends on:** D3 (sidecar deployment model), D4 (API contract)
**Status:** captured
