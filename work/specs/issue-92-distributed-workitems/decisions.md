# Decisions — Cross-Service WorkItem Federation (#95)

## D1: Federation scope — full coordination protocol in casehub-work

**Choice:** Full WS-HumanTask-inspired coordination protocol owned by casehub-work
**Alternatives:**
- Primitives only (CloudEvents, client module, transport SPI) — respects layering but doesn't solve end-to-end; requires Qhorus/CaseHub assembly
- Primitives + projection module — complete CQRS solution but no formal protocol; ad-hoc coordination
**Rationale:** Pre-release platform — design for the RIGHT architecture, not the easiest. The coordination protocol defines the formal contract between Task Parents and Task Processors, enabling clean bidirectional lifecycle coupling. The projection module provides local inbox performance. The client module enables lightweight integration.
**Trade-offs:** Most upfront work. Adds ceremony (protocol versioning, registration endpoints) that may need revision when CaseHub/Qhorus stabilize. But: having the protocol defined NOW means CaseHub and Qhorus integrate against a stable contract rather than evolving ad-hoc.
**Sources:**
- [WS-HumanTask 1.1 spec](https://docs.oasis-open.org/bpel4people/ws-humantask-1.1-spec-cs-01.html) — Task Parent / Task Processor / Task Client separation, coordination protocol
- [Camunda 8.8 Unified Architecture](https://camunda.com/blog/2025/11/whats-different-orchestration-cluster-camunda-88/) — industry moved to single-cluster; federation is custom
- [CQRS pattern (Azure)](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) — single-writer + read projections
- [CloudEvents spec](https://cloudevents.io/) — standard event envelope
- docs/LAYERING.md — casehub-work boundary rule
- GE-20260805-a2aa1b — WorkItemStore SPI co-location with JPA (lightweight client needed)
**Exploration:** deep-analysis
**Status:** captured

## D2: Federation scenario — multi-service mesh with per-interaction asymmetry

**Choice:** Multi-service mesh with per-interaction Task Parent / Task Processor roles. Any service running casehub-work can be both creator and resolver. Protocol is asymmetric per-WorkItem (one owner) but symmetric in capability.
**Alternatives:**
- AI agent → human inbox only — artificially constrains to one-directional; platform architecture naturally produces any-to-any
- Hierarchical delegation — tree structure is a subset of mesh; designing for mesh doesn't prevent tree usage
**Rationale:** Traced from the platform's own consumers: CaseHub engine, Qhorus agents, claudony dashboard, and standalone apps all produce the same pattern — any service creates WorkItems that may need resolution in any other service. Per-interaction asymmetry (single owner per WorkItem) provides clean consistency semantics. System-level symmetry means no service is special.
**Trade-offs:** Mesh is more complex than one-directional. Service discovery matters. But the alternatives would require redesign when the next consumer arrives.
**Sources:**
- [A2A protocol spec](https://a2a-protocol.org/latest/specification/) — "fluid roles" (per-interaction asymmetry, system symmetry), now Linux Foundation standard with 150+ org support
- [WS-HumanTask 1.1](https://docs.oasis-open.org/bpel4people/ws-humantask-1.1-spec-cs-01.html) — Task Parent / Task Processor / Task Client separation
- CaseHub engine A2A module (`engine/a2a/`) — A2AClient, AgentCard already exist
- Qhorus messaging infrastructure — webhook, Kafka, WebSocket, PostgreSQL observers already exist
- docs/NORMATIVE-ALIGNMENT.md — WorkItem lifecycle ↔ Qhorus speech act mapping
- docs/LAYERING.md — ecosystem context (casehub-work below CaseHub, Qhorus, Quarkus-Flow)
**Depends on:** D1
**Exploration:** deep-analysis
**Status:** captured

## D3: Protocol relationship — casehub-work protocol independent, Qhorus as transport, A2A as discovery

**Choice:** casehub-work defines its own WorkItem-specific coordination protocol (CloudEvents-based). Dual transport: REST carries synchronous commands (mutations requiring confirmation), Qhorus/CloudEvents carries asynchronous lifecycle events. A2A compatibility is a separate adapter module.
**Alternatives:**
- Protocol IS an A2A profile/extension — A2A's 8 task states are too simple for WorkItem's 15+ lifecycle states; domain-specific operations (claim with OCC, delegate, escalate) don't map to A2A's generic messages
- Direct REST only (no protocol layer) — works for simple cases but no lifecycle coupling, no projection model, no subscription semantics
- Single transport (Qhorus only) — commands need synchronous confirmation (operator must know if their claim succeeded); Qhorus channels are asynchronous by design
**Rationale:** casehub-work needs domain-specific protocol messages (claim, delegate, escalate, SLA breach, group completion) that A2A can't express. But A2A provides excellent discovery (Agent Cards) and the engine already has A2A infrastructure. Clean separation: casehub-work defines WHAT (protocol messages and semantics), REST provides HOW for commands (synchronous, confirmation-required), Qhorus/CloudEvents provides HOW for events (asynchronous, fire-and-observe), A2A provides WHERE (discovery), CaseHub provides WHY (orchestration).
**Trade-offs:** Multiple layers to understand. But each layer has a clear owner and can evolve independently. A2A interop requires a separate adapter, adding a module.
**Sources:**
- [A2A protocol spec](https://a2a-protocol.org/latest/specification/) — 8 task states vs casehub-work's 15+; generic message model vs domain-specific operations
- [CloudEvents spec](https://cloudevents.io/) — CNCF graduated standard for event envelope
- Qhorus runtime — MessageService, MessageObserverDispatcher, multiple delivery backends
- CaseHub engine — A2AClient, AgentCard, A2AWorkerFunction
- WorkCloudEventAdapter, WorkCloudEventInboundAdapter — CloudEvents infrastructure already exists
**Depends on:** D1, D2
**Exploration:** deep-analysis
**Status:** revised — clarified dual-transport model: REST for synchronous commands, Qhorus/CloudEvents for asynchronous lifecycle events. Previous framing ("Qhorus as primary transport") was inconsistent with D6's REST proxy design.

## D4: Projection model — shadow WorkItems as full WorkItemEntity rows with origin marker

**Choice:** Add `originServiceId` (nullable String) and `originWorkItemId` (nullable UUID) to `WorkItemEntity`. Shadows are real rows in the same table. All existing inbox queries, filter engine, queue membership, and reports work transparently.
**Alternatives:**
- Separate `FederatedWorkItemView` entity — clean separation, but requires duplicating inbox query logic, filter engine integration, queue membership, and reports (4x integration work). 35 chapters of query/filter refinement would need parallel maintenance.
**Rationale:** A shadow IS a WorkItem from the domain perspective. The operator doesn't care about provenance — they claim, work, complete. Using the same entity maximizes code reuse and minimizes new surface. Mutation risk is mitigated by service-layer guards (`if (originServiceId != null) route to proxy`), timer/scheduler skip guards, and assignment strategy skip guards.
**Trade-offs:** Risk of accidental local mutation mitigated by `FederationGuardStore` — a CDI `@Decorator` on `WorkItemStore` that rejects `put()` calls on shadow WorkItems (where `originServiceId != null`) unless `FederationSyncContext` (ThreadLocal) is active. This is a persistence-boundary guard that catches ALL mutation paths — REST, CDI, timers, schedulers, assignment strategies — with one guard instead of 30+ scattered service-layer checks. The REST-level `FederationProxy` (D6) intercepts operator-initiated mutations before reaching the service layer; the store-level guard is the safety net for everything else. Shadow WorkItems have many irrelevant nullable fields (spawn group config, SLA policy config). But WorkItem already has 50+ fields — two more is consistent with the existing pattern.
**Sources:**
- WorkItemStore SPI (api/) — `put()` is the single persistence boundary for all mutations
- WorkItemService (runtime/) — 30 public methods, all mutations route through `WorkItemStore.put()`
- ExpiryLifecycleService — expireItem/processClaimDeadline also route through `WorkItemStore.put()`
- WorkItemAssignmentService — auto-assignment routes through `WorkItemStore.put()`
- CDI `@Decorator` pattern — standard mechanism for intercepting and conditionally delegating
- docs/MODULES.md — 35+ chapters of query/filter refinement
**Depends on:** D1, D2, D3, D5
**Exploration:** deep-analysis (first-principles, traced from codebase complexity)
**Status:** captured

## D5: Subscription model — filter-on-creation with full lifecycle tracking

**Choice:** Service B registers a subscription with Service A providing peer ID, callback URL, and filter predicate (candidateGroups, candidateUsers, tenancyId). Filter is evaluated only on WorkItem CREATION. Once matched, the subscription "locks on" and delivers ALL subsequent lifecycle events for that WorkItem until terminal state. Subscriptions are persistent resources stored in the federation module.
**Alternatives:**
- Topic-based subscription (by group/type) — efficient but rigid; groups change, requires topic management
- Full stream + local filter — simplest but wastes bandwidth; O(all events) per subscriber
- Push creation + pull on demand (lazy projection) — interesting but adds first-access latency
**Rationale:** candidateGroups/Users are the natural partition keys. Filter-on-creation is O(creations × subscribers × filter) rather than O(all events × all subscribers × filter). Once locked, full lifecycle delivery ensures shadow stays current without re-evaluation. Same model as SSE filtering (WorkItemEventBroadcaster) but cross-service. This is NOT a duplication of the platform subscription engine (#331). The platform `Subscription` is a notification routing system (targets: Slack, Teams, webhook; per-event stateless filter evaluation via `ExpressionEvaluator`; `NotificationTemplate` formatting). Federation subscriptions are data replication (targets: peer services; filter-on-creation with per-WorkItem lifecycle lock-on; full WorkItem state snapshots for shadow projection). The platform engine has no concept of per-entity subscription tracking — it evaluates filters independently against each event.
**Trade-offs:** Subscriptions need persistence and restart recovery. Filter predicate language needs definition (simple equality / set-contains is enough initially). Late-joining subscribers miss already-created WorkItems (solvable with catch-up query on registration).
**Sources:**
- WorkItemEventBroadcaster SPI — existing filtered SSE model (by workItemId, type, tenancyId)
- WorkItemEntity.candidateGroups / candidateUsers — natural partition keys
**Depends on:** D2, D3
**Exploration:** deep-analysis
**Status:** captured

## D6: Error handling — fail-fast with configurable timeout

**Choice:** FederationProxy calls owner REST API synchronously with configurable timeout (default 5s). On success: shadow updated from CloudEvent response. On failure: error returned to operator. On 409: shadow updated from next CloudEvent, operator notified. No new lifecycle states, no background retry, no reconciliation.
**Alternatives:**
- Tentative claim (CLAIM_PENDING state + background retry) — better UX for transient failures but adds lifecycle complexity and reconciliation logic
- Local-first with reconciliation — most available but risks conflicting state; violates single-writer axiom
- Queue and retry (accept locally, retry async) — risk of optimistic conflict if someone else claims on owner
**Rationale:** Owner is single source of truth. A claim is only valid when the owner confirms it. Transient failures at human scale (seconds) are tolerable — operator retries manually. Transport-layer reliability (Qhorus delivery guarantees, Kafka acknowledgment) handles event delivery; command reliability is solved by timeout + retry. No new states means no lifecycle change for federation.
**Trade-offs:** Operator sees errors during owner downtime. But: errors are honest — better than tentative states that might revert. Future enhancement: Qhorus channel transport provides retry/dead-letter without custom logic. Staleness window: between a 409/timeout response and the arrival of the corrective CloudEvent, the shadow's displayed state is stale (e.g. shows PENDING when owner is ASSIGNED). Duration bounded by CloudEvent delivery latency (sub-second in Qhorus, seconds in webhook). UI should communicate uncertainty: "Claim failed — another actor may have claimed this item. Status will update shortly."
**Sources:**
- WorkItemService.claim() — OCC with @Version; can't work cross-service without single writer
- [A2A protocol](https://a2a-protocol.org/latest/specification/) — async task submission model (submitted → working)
**Depends on:** D1, D3, D4
**Exploration:** deep-analysis
**Status:** captured

## D7: Audit trail spanning — dual audit with cross-references

**Choice:** Both services maintain audit entries. Owner has authoritative trail (all mutations go through owner's WorkItemService). Shadow has local trail recording proxied operations with federation metadata (originServiceId, originWorkItemId, remoteAuditEntryId). CloudEvents carry audit entry IDs in extension attributes for cross-referencing. ProvenanceLink (#39) connects them into a unified PROV-O causal graph when it lands.
**Alternatives:**
- Owner-only audit — simple but Service B can't answer "what did our operators do?" without querying every remote owner
- Federated audit events (replicate owner's full audit trail to shadows) — complete but duplicates data and requires audit-specific sync
**Rationale:** Neither trail is redundant: owner has the canonical record of what happened, shadow has the local operator activity log. ProvenanceLink is the planned cross-service traceability layer — dual audit with cross-references lays the foundation. Local audit enables independent compliance review per service.
**Trade-offs:** Two audit trails for one logical WorkItem. Cross-reference integrity depends on CloudEvent delivery. ProvenanceLink (#39) dependency for full unified view. Hash chain integrity is per-service only — each service's Merkle chain provides independent tamper-evidence within its boundary. Cross-service audit integrity is pointer-based (ProvenanceLink / `remoteAuditEntryId`), not cryptographic. An auditor verifying cross-service integrity must check both chains independently; the cross-reference is a pointer, not a cryptographic link.
**Sources:**
- AuditEntryStore (runtime/) — existing per-WorkItem audit trail
- casehub-work-ledger — hash chain integrity for audit entries
- Issue #39 (ProvenanceLink) — PROV-O causal graph for cross-service chains
**Depends on:** D3, D4
**Exploration:** deep-analysis
**Status:** captured

## D8: Protocol evolution — capability-based negotiation

**Choice:** Peers advertise supported operations (claim, delegate, complete, escalate) and supported event types (created, assigned, completed, expired) at subscription registration. New operations and event types are additive — peers that don't advertise them don't receive them. Semantically changed operations get new names rather than version bumps. Receivers tolerate unknown fields (`@JsonIgnoreProperties(ignoreUnknown = true)`) and unknown event types (log + skip). CloudEvents `specversion` handles envelope versioning.
**Alternatives:**
- Semantic versioning (protocolVersion `1.0.0`) — well-understood but conflates format and semantic compatibility; major version bumps force O(N²) re-registration across a mesh of N services; adds a second version axis on top of CloudEvents specversion
- CloudEvents type versioning (e.g., `io.casehub.work.created.v1`) — version per event type adds granularity but complicates routing
**Rationale:** Capability-based negotiation avoids the big-bang coordination cost of major version bumps. New operations are additive — old peers don't advertise them, don't receive them, no disruption. Builds on the existing Capability vocabulary type (C34, CapabilityRegistry SPI). Fills the discovery gap when A2A is absent — peers advertise capabilities at registration rather than relying on Agent Cards from an optional A2A adapter. Format compatibility already handled by `@JsonIgnoreProperties(ignoreUnknown = true)` and log-skip for unknown event types.
**Trade-offs:** Capability negotiation adds per-registration validation (verify supported operations overlap). But this is O(1) per registration, not O(N²) per version bump. Semantic changes require new operation names rather than version bumps — a naming discipline, but one that forces explicit breakage.
**Sources:**
- [CloudEvents spec](https://cloudevents.io/) — specversion field, extensibility model
- WorkCloudEventAdapter — existing event type naming (`io.casehub.work.lifecycle`)
- CapabilityRegistry SPI (C34) — existing capability vocabulary pattern
- [HTTP evolution model](https://www.rfc-editor.org/rfc/rfc7231) — additive methods and headers, no semantic version axis
**Depends on:** D3
**Exploration:** quick (revised after reviewer challenge R1-04)
**Status:** revised — changed from semantic versioning to capability-based negotiation. Semver conflates format and semantic compatibility, forces big-bang coordination across the mesh, and adds a redundant version axis on top of CloudEvents specversion.

## D9: Federation module placement — optional integration module

**Choice:** Federation lives in a single `federation/` optional integration module at the same build position as `engine-adapter/` and `qhorus/`. Federation fields (`originServiceId`, `originWorkItemId`) are added to the `WorkItem` record in `api/` (see D12). The `federation/` module contains: `FederationGuardStore` decorator, `FederationProxy` service, subscription entities (JPA), CloudEvent inbound handler, REST endpoints for subscription registration and proxy routing.
**Alternatives:**
- Multi-module split (`federation-api/`, `federation-runtime/`) — follows three-tier pattern but federation has no consumer-facing SPI that needs to be in a separate pure-Java module; the guard decorator and proxy are implementation concerns
- Inline in `runtime/` — minimal module count but violates the optional-module principle; non-federating deployments would carry federation code
**Rationale:** Federation is an integration concern, like `engine-adapter/` and `qhorus/`. It depends on `api/` (WorkItem record with federation fields), `runtime/` (WorkItemStore SPI implementations, WorkItemService), and `rest/` (proxy REST endpoints). Non-federating deployments exclude it from the classpath — zero overhead. The `FederationGuardStore` CDI `@Decorator` activates only when the module is present.
**Trade-offs:** Single module means JPA entities, REST endpoints, and CDI beans are co-located. This is consistent with other integration modules (`engine-adapter/` has entities, CDI beans, and event observers in one module).
**Sources:**
- ARC42STORIES.MD §5 — Module Index, existing module taxonomy
- `engine-adapter/` — precedent for integration module structure
- `qhorus/` — precedent for optional protocol bridge module
**Depends on:** D1, D4, D12
**Exploration:** surfaced by reviewer (R1-09)
**Status:** captured

## D10: Service-to-service security — platform identity tokens with HMAC event signing

**Choice:** Peer services authenticate via platform identity tokens (`casehub-platform-identity` module) for REST command calls (D6) and subscription registration (D5). Inbound CloudEvents carry HMAC signatures verified against a shared secret established at subscription registration time. Peer trust is established during subscription registration via mutual token exchange.
**Alternatives:**
- mTLS only — transport-level trust, no application-level identity; doesn't cover CloudEvent integrity for webhook-based delivery
- Custom federation-specific trust model — unnecessary when platform identity module already provides service credentials
- No authentication (deferred) — explicit insecurity is worse than simple security
**Rationale:** The platform identity module already provides service-to-service authentication. Federation extends this existing capability rather than building a parallel trust model. HMAC signing for CloudEvents adds integrity verification at the application level — transport-level TLS protects the channel, HMAC protects the payload against spoofing in webhook-based delivery where the transport endpoint is a URL the subscriber controls.
**Trade-offs:** HMAC shared secrets require secure distribution at registration time. Pairwise secrets in a mesh of N services means N×(N-1)/2 secrets — manageable at platform scale (single digits of services).
**Sources:**
- `casehub-platform-identity` — existing service identity module
- [CloudEvents webhook spec](https://github.com/cloudevents/spec/blob/main/cloudevents/adapters/webhook.md) — webhook validation patterns
**Depends on:** D3, D5, D6
**Exploration:** surfaced by reviewer (R1-10)
**Status:** captured

## D11: Multi-tenancy coherence — tenant-scoped federation with cross-tenant prohibition

**Choice:** Federation subscriptions are tenant-scoped. `tenancyId` is a mandatory filter field on subscription registration — a peer service subscribes to WorkItems within a specific tenant only. Cross-tenant federation (Service A in tenant X creating shadows in Service B's tenant Y) is prohibited by default. Shadow WorkItems inherit the owner's `tenancyId` and are subject to the same RLS policy (`rls/` package) as locally-created WorkItems.
**Alternatives:**
- Cross-tenant allowed — enables multi-tenant federation but has compliance implications; RLS boundary violations possible
- Tenant-agnostic (platform-global scope like `PLATFORM_TENANT_ID`) — simplest but ignores tenant boundaries entirely; the `WorkItemSubscriptionBridge` uses `PLATFORM_TENANT_ID` for the platform notification DataSource, but federation subscriptions need tenant isolation
**Rationale:** Row-level security is a foundational guarantee in casehub-work. Federation must not weaken it. Tenant-scoped subscriptions ensure that shadows respect the same RLS boundaries as locally-created WorkItems. The multi-tenancy design spec (issue-256) and RLS policy applicators remain coherent — federation adds no new tenant-crossing paths.
**Trade-offs:** Prevents cross-tenant federation scenarios. If a future use case requires cross-tenant (e.g., a central admin service viewing WorkItems across tenants), a dedicated cross-tenant subscription type with explicit opt-in can be added.
**Sources:**
- `rls/` package (runtime/) — existing row-level security
- WorkItemSubscriptionBridge — uses `PLATFORM_TENANT_ID` for notification DataSource
- Issue #256 — multi-tenancy design spec
**Depends on:** D5
**Exploration:** surfaced by reviewer (R1-11)
**Status:** captured

## D12: WorkItem SPI federation fields — on the `WorkItem` record in `api/`

**Choice:** `originServiceId` (nullable String) and `originWorkItemId` (nullable UUID) are added to the `WorkItem` record in `api/`. All three `WorkItemStore` backends (JPA via `WorkItemEntity`, MongoDB via document schema, InMemory via `WorkItem` record) persist these fields. Null for locally-created WorkItems.
**Alternatives:**
- JPA-only (`WorkItemEntity` column, not on `WorkItem` record) — clean separation but the `FederationGuardStore` decorator (D4) operates on `WorkItem` through the SPI and needs `originServiceId` to detect shadows. Also prevents consumer code from detecting shadows through the API.
- Separate `FederatedWorkItemView` type — requires parallel query paths (rejected in D4 for the same reason)
**Rationale:** The `FederationGuardStore` decorator (D4) intercepts `WorkItemStore.put(WorkItem)` and checks `item.originServiceId() != null`. This requires the field on the `WorkItem` record, not just the JPA entity. All backends must support the fields for the store SPI contract to be uniform. The `WorkItem` record already has 50+ fields, many nullable — two more nullable fields is consistent with the existing pattern.
**Trade-offs:** Every consumer of the `WorkItem` type (`casehub-engine`, `casehub-clinical`, `casehub-devtown`) sees the federation fields. This is intentional — any code handling WorkItems should be able to detect shadows. The `WorkItem.Builder` gains two more setter methods.
**Sources:**
- WorkItem record (api/) — existing 50+ field record with builder
- WorkItemStore SPI (api/) — `put(WorkItem)` contract
- JpaWorkItemStore, MongoWorkItemStore, InMemoryWorkItemStore — three backend implementations
**Depends on:** D4
**Exploration:** surfaced by reviewer (R1-12)
**Status:** captured
