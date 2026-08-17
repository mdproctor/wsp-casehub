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

**Choice:** casehub-work defines its own WorkItem-specific coordination protocol (CloudEvents-based). Qhorus channels are the primary transport. A2A compatibility is a separate adapter module.
**Alternatives:**
- Protocol IS an A2A profile/extension — A2A's 8 task states are too simple for WorkItem's 15+ lifecycle states; domain-specific operations (claim with OCC, delegate, escalate) don't map to A2A's generic messages
- Direct REST only (no protocol layer) — works for simple cases but no lifecycle coupling, no projection model, no subscription semantics
**Rationale:** casehub-work needs domain-specific protocol messages (claim, delegate, escalate, SLA breach, group completion) that A2A can't express. But A2A provides excellent discovery (Agent Cards) and the engine already has A2A infrastructure. Clean separation: casehub-work defines WHAT (protocol), Qhorus provides HOW (transport), A2A provides WHERE (discovery), CaseHub provides WHY (orchestration).
**Trade-offs:** Multiple layers to understand. But each layer has a clear owner and can evolve independently. A2A interop requires a separate adapter, adding a module.
**Sources:**
- [A2A protocol spec](https://a2a-protocol.org/latest/specification/) — 8 task states vs casehub-work's 15+; generic message model vs domain-specific operations
- [CloudEvents spec](https://cloudevents.io/) — CNCF graduated standard for event envelope
- Qhorus runtime — MessageService, MessageObserverDispatcher, multiple delivery backends
- CaseHub engine — A2AClient, AgentCard, A2AWorkerFunction
- WorkCloudEventAdapter, WorkCloudEventInboundAdapter — CloudEvents infrastructure already exists
**Depends on:** D1, D2
**Exploration:** deep-analysis
**Status:** captured

## D4: Projection model — shadow WorkItems as full WorkItemEntity rows with origin marker

**Choice:** Add `originServiceId` (nullable String) and `originWorkItemId` (nullable UUID) to `WorkItemEntity`. Shadows are real rows in the same table. All existing inbox queries, filter engine, queue membership, and reports work transparently.
**Alternatives:**
- Separate `FederatedWorkItemView` entity — clean separation, but requires duplicating inbox query logic, filter engine integration, queue membership, and reports (4x integration work). 35 chapters of query/filter refinement would need parallel maintenance.
**Rationale:** A shadow IS a WorkItem from the domain perspective. The operator doesn't care about provenance — they claim, work, complete. Using the same entity maximizes code reuse and minimizes new surface. Mutation risk is mitigated by service-layer guards (`if (originServiceId != null) route to proxy`), timer/scheduler skip guards, and assignment strategy skip guards.
**Trade-offs:** Risk of accidental local mutation (mitigated by guards). Shadow WorkItems have many irrelevant nullable fields (spawn group config, SLA policy config). But WorkItem already has 15+ nullable fields — two more is consistent with the existing pattern.
**Sources:**
- WorkItemService (runtime/) — 40+ methods, each needs a federation guard
- FilterRegistryEngine (runtime/filter/) — complex filter chain, reused for shadows
- WorkItemTimerService — skip shadows for local timer scheduling
- WorkItemAssignmentService — skip shadows for local assignment strategies
- docs/MODULES.md — 35+ chapters of query/filter refinement
**Depends on:** D1, D2, D3
**Exploration:** deep-analysis (first-principles, traced from codebase complexity)
**Status:** captured

## D5: Subscription model — filter-on-creation with full lifecycle tracking

**Choice:** Service B registers a subscription with Service A providing peer ID, callback URL, and filter predicate (candidateGroups, candidateUsers, tenancyId). Filter is evaluated only on WorkItem CREATION. Once matched, the subscription "locks on" and delivers ALL subsequent lifecycle events for that WorkItem until terminal state. Subscriptions are persistent resources stored in the federation module.
**Alternatives:**
- Topic-based subscription (by group/type) — efficient but rigid; groups change, requires topic management
- Full stream + local filter — simplest but wastes bandwidth; O(all events) per subscriber
- Push creation + pull on demand (lazy projection) — interesting but adds first-access latency
**Rationale:** candidateGroups/Users are the natural partition keys. Filter-on-creation is O(creations × subscribers × filter) rather than O(all events × all subscribers × filter). Once locked, full lifecycle delivery ensures shadow stays current without re-evaluation. Same model as SSE filtering (WorkItemEventBroadcaster) but cross-service.
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
**Trade-offs:** Operator sees errors during owner downtime. But: errors are honest — better than tentative states that might revert. Future enhancement: Qhorus channel transport provides retry/dead-letter without custom logic.
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
**Trade-offs:** Two audit trails for one logical WorkItem. Cross-reference integrity depends on CloudEvent delivery. ProvenanceLink (#39) dependency for full unified view.
**Sources:**
- AuditEntryStore (runtime/) — existing per-WorkItem audit trail
- casehub-work-ledger — hash chain integrity for audit entries
- Issue #39 (ProvenanceLink) — PROV-O causal graph for cross-service chains
**Depends on:** D3, D4
**Exploration:** deep-analysis
**Status:** captured

## D8: Protocol versioning — semantic versioning with additive-only minor changes

**Choice:** protocolVersion (e.g., `1.0.0`) included in subscription registration. Within major version: new event types and new fields are additive-only (non-breaking). Major version change: peers must re-register; old subscriptions rejected with descriptive error. Receivers tolerate unknown fields (`@JsonIgnoreProperties(ignoreUnknown = true)`) and unknown event types (log + skip). Feature discovery via Agent Cards when A2A adapter is present.
**Alternatives:**
- CloudEvents type versioning (e.g., `io.casehub.work.created.v1`) — version per event type adds granularity but complicates routing
- Feature flags (peers advertise capabilities) — flexible but adds negotiation complexity
**Rationale:** CloudEvents + JSON already provide forward compatibility via unknown-field tolerance. Semantic versioning is well-understood. Protocol evolution follows HTTP API evolution patterns: additive within major, breaking across major. protocolVersion in registration enables clean rejection of incompatible peers.
**Trade-offs:** Major version bumps are disruptive (all peers must re-register). But: pre-release platform — breaking changes cost nothing. Additive-only within major version is a design discipline, not a technology constraint.
**Sources:**
- [CloudEvents spec](https://cloudevents.io/) — specversion field, extensibility model
- WorkCloudEventAdapter — existing event type naming (`io.casehub.work.lifecycle`)
**Depends on:** D3
**Exploration:** quick
**Status:** captured
