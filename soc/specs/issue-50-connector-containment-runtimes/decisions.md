# Decisions — Connector-Based Containment Runtimes (#50)

## D1: Self-contained SOC endpoint resolver (no workers-http dependency)

**Choice:** SOC owns its own `ContainmentEndpointResolver` — a simple `@ApplicationScoped` bean that reads `casehub.soc.containment.endpoints.*` config at `@PostConstruct`. No compile dependency on the workers repo. HTTP calls via Vert.x WebClient directly.
**Alternatives:**
- Workers HTTP config infrastructure — inject `HttpEndpointResolver` for endpoint resolution. Incompatible: `initialize()` is package-private, called by `HttpWorkerRuntime` during the worker runtime lifecycle, not at `@PostConstruct`. Injecting from SOC gives an empty endpoint map.
- Full dispatch pipeline — route through `HttpWorkerExecutionManager.submit()`. Incompatible: fire-and-forget async, cannot return synchronous `ContainmentResult`.
**Rationale:** SOC's containment endpoints are static config — a simple config reader is sufficient. The workers repo's 3-tier resolution (SPI routes → config → EndpointRegistry) is more infrastructure than SOC needs, and the lifecycle coupling makes it impractical to use as a library. Self-contained means no cross-repo dependency and no initialization order concerns.
**Trade-offs:** Config loading logic (~30 lines) is duplicated from `HttpEndpointResolver.loadConfigEndpoints()` pattern. Acceptable — the pattern is trivial, and the alternative (depending on a module with lifecycle coupling) is worse.
**Sources:** `HttpEndpointResolver.java` (reference for config pattern, not a dependency), `HttpWorkerExecutionManager.submitSync()` (reference for HTTP call pattern, not a dependency)
**Exploration:** quick
**Status:** revised (post-spec review: HttpEndpointResolver lifecycle incompatibility)

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
**Trade-offs:** Two config layers: (1) action type → endpoint tag (`casehub.soc.containment.routing.isolate-host=crowdstrike`), (2) endpoint tag → URL (`casehub.workers.http.endpoints.crowdstrike.url=https://...`). Layer 1 is SOC-specific config. Layer 2 reuses the workers HTTP endpoint config system. Both must be present for an action to route to an external connector. Unmapped action types in layer 1 fall through to LoggingContainmentExecutor.
**Sources:** `SocActionType.java` (action types), `HttpEndpointResolver.loadConfigEndpoints()` (config pattern for layer 2)
**Exploration:** quick
**Status:** revised (decision review: clarified two-layer config)

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

## D10: Action type to integration mapping

**Choice:** The 9 `SocActionType` values map to 3 integrations plus 1 internal:

| Action type | Integration | Batch |
|---|---|---|
| `ENABLE_ENHANCED_LOGGING` | Internal (no external connector — logging config change) | 1 (simulated) |
| `ROTATE_API_KEY` | Okta/Azure AD | 4 |
| `BLOCK_IP` | Palo Alto | 3 |
| `BLOCK_DOMAIN` | Palo Alto | 3 |
| `DISABLE_USER_ACCOUNT` | Okta/Azure AD | 4 |
| `ISOLATE_HOST` | CrowdStrike Falcon | 2 |
| `REVOKE_CREDENTIALS` | Okta/Azure AD | 4 |
| `NETWORK_SEGMENTATION` | Palo Alto | 3 |
| `WIPE_ENDPOINT` | CrowdStrike Falcon | 2 |

Three connectors total: CrowdStrike (2 actions), Palo Alto (3 actions), Okta (3 actions). `ENABLE_ENHANCED_LOGGING` is fully reversible, no gate, and can remain internal (LoggingContainmentExecutor handles it correctly since it's a logging config change, not an external API call).
**Alternatives:**
- Fewer integrations — merge Palo Alto network actions into CrowdStrike (some EDR platforms handle both). Less realistic but fewer connectors.
- More integrations — separate DNS blocking (BLOCK_DOMAIN) from firewall blocking (BLOCK_IP). More granular but premature.
**Rationale:** Groups by external system responsibility. CrowdStrike handles endpoint actions (isolate, wipe). Palo Alto handles network actions (block IP, block domain, segment). Okta/Azure AD handles identity actions (disable account, revoke credentials, rotate keys). This matches how real SOC deployments are typically structured.
**Trade-offs:** `ENABLE_ENHANCED_LOGGING` not going through a connector means it can't be externally managed. Acceptable — it's the only fully reversible, never-gated action.
**Sources:** `SocActionType.java` (9 action types with gate policies and reversibility)
**Exploration:** quick
**Depends on:** D4 (per-integration endpoints), D8 (batch ordering)
**Status:** captured
