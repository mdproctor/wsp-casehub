# Cross-Service WorkItem Federation — Design Spec

**Issue:** #95 (epic #92)
**Date:** 2026-08-18
**Status:** Draft
**Decisions:** D1–D12 (validated via standard review, 3 rounds, 13 issues, 12 resolved)

---

## 1. Problem

casehub-work assumes all WorkItem lifecycle operations occur against a single database within a single JVM cluster. A Qhorus AI agent in Service A cannot create a WorkItem that appears in Service B's human operator inbox for claim and resolution.

Single-cluster distribution (shared database, PostgreSQL LISTEN/NOTIFY for SSE, `@Version` OCC for atomic claims) is already done (#93, #94, #96, #155). This spec addresses **cross-service federation**: WorkItems created in one service, resolved in another, with different databases.

## 2. Design Principles

1. **Single-writer ownership.** Each WorkItem has one owner — the service whose database it lives in. Distributed mutable state is a consistency disaster for entities with 15+ lifecycle transitions, OCC guards, SLA timers, and audit trails.

2. **Events out, commands in (CQRS).** Lifecycle events flow outbound via CloudEvents. Mutations flow inbound via REST API. The owning service is the single write model; consuming services maintain read-only projections.

3. **Per-interaction asymmetry, system-level symmetry.** Per WorkItem, roles are clear: one Task Processor (owner), zero or more Task Clients (consumers). But any service can play either role — the protocol is symmetric in capability.

4. **Transport-agnostic.** The protocol defines WHAT messages are exchanged and their semantics. HOW they travel (webhook, Qhorus channel, Kafka) is pluggable.

5. **Layering.** casehub-work provides federation primitives and coordination protocol. CaseHub orchestrates (when to federate). Qhorus transports (how events travel). A2A discovers (which services exist).

## 3. Architecture Overview

```
Service A (owner)                                Service B (consumer)
+-----------------------------------------+      +-----------------------------------------+
| WorkItemService                         |      |                                         |
|   create() -> store.put()               |      |                                         |
|   -> WorkCloudEventAdapter              |      |                                         |
|     -> FederationEventRouter            |      |                                         |
|       -> evaluate subscriptions         |      |                                         |
|         -> FederationTransport.send() --+--CE->+-- FederationReceiver                   |
|                                         |      |     -> set FederationSyncContext         |
|                                         |      |     -> store.put(shadow WorkItem)        |
|                                         |      |     -> clear FederationSyncContext        |
|                                         |      |     -> SSE broadcast to local clients    |
|                                         |      |                                         |
|                                         |      | Operator claims shadow:                  |
|                                         |      |   WorkItemResource.claim()               |
| REST API <------------------------------+--REST+--<- FederationProxy.claim()             |
|   WorkItemService.claim()               |      |     (originServiceId != null)            |
|   -> store.put() (ASSIGNED)             |      |                                         |
|   -> WorkCloudEventAdapter              |      |                                         |
|     -> FederationTransport.send() ------+--CE->+-- FederationReceiver                   |
|                                         |      |     -> update shadow to ASSIGNED         |
+-----------------------------------------+      +-----------------------------------------+

CE = CloudEvent via FederationTransport (webhook, Qhorus, Kafka)
REST = synchronous HTTP to owner's REST API
```

## 4. New Modules

| Module | Type | Purpose |
|--------|------|---------|
| `federation/` | Optional integration module | `FederationGuardStore`, `FederationProxy`, `FederationReceiver`, `FederationEventRouter`, subscription entities, REST endpoints |
| `client/` | Lightweight REST client | HTTP client for remote WorkItem operations. No JPA, no CDI. Used by `FederationProxy` and standalone consumers |

Both modules are activated by classpath presence, consistent with `engine-adapter/` and `qhorus/`. Non-federating deployments exclude them — zero overhead.

## 5. WorkItem Record Changes (D12)

Two nullable fields added to the `WorkItem` record in `api/`:

| Field | Type | Null means | Non-null means |
|-------|------|------------|----------------|
| `originServiceId` | `String` | Locally owned | Shadow — owned by this service |
| `originWorkItemId` | `UUID` | Locally owned | ID on the owning service |

All three `WorkItemStore` backends (JPA, MongoDB, InMemory) persist these fields. The `WorkItem.Builder` gains two setter methods. Flyway migration adds two nullable columns to the `work_item` table.

Consumer code can detect shadows: `if (workItem.originServiceId() != null) { /* shadow */ }`.

## 6. Federation Guard (D4)

`FederationGuardStore` is a CDI `@Decorator` on `WorkItemStore`:

```
@Decorator @Priority(Interceptor.Priority.APPLICATION)
class FederationGuardStore implements WorkItemStore {

    @Delegate WorkItemStore delegate;

    WorkItem put(WorkItem item) {
        if (item.originServiceId() != null
                && !FederationSyncContext.isActive()) {
            throw new FederatedWorkItemMutationException(item.id());
        }
        return delegate.put(item);
    }

    // all other methods delegate transparently
}
```

`FederationSyncContext` is a `ThreadLocal<Boolean>` set only by `FederationReceiver` when processing inbound CloudEvents. This provides a single persistence-boundary guard that catches ALL mutation paths — WorkItemService, ExpiryLifecycleService, WorkItemAssignmentService, WorkItemSpawnService, timers, schedulers — with one guard instead of 30+ scattered checks.

The `@Decorator` activates only when the `federation/` module is on the classpath.

## 7. Coordination Protocol (D3, D8)

### Dual Transport

| Channel | Carries | Transport | Why |
|---------|---------|-----------|-----|
| **Commands** | claim, complete, reject, delegate, escalate, release, suspend, resume, cancel | REST (synchronous) | Operator needs confirmation that the command succeeded |
| **Events** | created, assigned, claimed, started, completed, rejected, expired, escalated, delegated, suspended, resumed, cancelled, faulted, obsoleted | CloudEvents (asynchronous) | Fire-and-observe — projection updates, no confirmation needed |

### Event Envelope

All federation events are CloudEvents (CNCF graduated standard). Event types follow existing naming:

```
type:       io.casehub.work.federation.<event-type>
source:     urn:casehub:work:<service-id>
subject:    <work-item-id>
datacontenttype: application/json
Extensions:
  tenancyid:      <tenant-id>
  auditentryid:   <audit-entry-id on owner>
```

Event data is the full `WorkItem` state snapshot (not a delta). Receivers overwrite the shadow's state from the snapshot. This is simpler than delta-based sync and tolerant of missed events — any single event brings the shadow fully current.

### Capability-Based Negotiation (D8)

At subscription registration, peers advertise:
- **Supported operations:** `[claim, complete, reject, delegate, escalate, ...]`
- **Supported event types:** `[created, assigned, completed, expired, ...]`

New operations and event types are additive. Peers that don't advertise them don't receive them. Semantically changed operations get new names rather than version bumps. Receivers tolerate unknown fields (`@JsonIgnoreProperties(ignoreUnknown = true)`) and unknown event types (log + skip).

## 8. Subscription Model (D5)

### Registration

Service B registers with Service A:

```
POST /federation/subscriptions
{
  "peerId": "service-b",
  "callbackUrl": "https://service-b.example.com/federation/events",
  "tenancyId": "tenant-1",
  "filter": {
    "candidateGroups": ["legal-review", "compliance"],
    "candidateUsers": []
  },
  "capabilities": {
    "operations": ["claim", "complete", "reject", "delegate"],
    "eventTypes": ["created", "assigned", "completed", "rejected", "expired"]
  },
  "hmacSecret": "<base64-encoded-shared-secret>"
}
```

Returns a persistent `FederationSubscription` resource with an ID.

### Filter-on-Creation with Lifecycle Lock-On

1. WorkItem is created on Service A
2. `FederationEventRouter` evaluates the creation event against all active subscriptions
3. If the WorkItem's `candidateGroups` intersects the subscription's filter: the subscription **locks on** to this WorkItem
4. ALL subsequent lifecycle events for this WorkItem are delivered to Service B until terminal state (COMPLETED, REJECTED, CANCELLED, EXPIRED, OBSOLETE, FAULTED)
5. No re-evaluation of the filter on subsequent events

Lock-on tracking is stored in a `federation_subscription_tracking` table: `(subscription_id, work_item_id)`.

### Catch-Up on Registration

When a new subscription is registered, Service A queries for existing non-terminal WorkItems matching the filter and sends a synthetic `created` event for each. This prevents late-joining subscribers from missing already-created WorkItems.

## 9. Federation Proxy (D6)

When a write operation targets a shadow WorkItem (`originServiceId != null`), the REST resource layer intercepts and routes to `FederationProxy`:

```
WorkItemResource.claim(id, actor) {
    WorkItem item = service.findById(id);
    if (item.originServiceId() != null) {
        return federationProxy.claim(item, actor);
    }
    return service.claim(id, actor);
}
```

`FederationProxy` uses the `client/` module to call the owner's REST API:

```
PUT https://<owner-url>/workitems/<originWorkItemId>/claim?claimant=<actor>
```

- **Timeout:** configurable (default 5 seconds)
- **On success:** owner processes the claim and emits a CloudEvent. The shadow is updated when the event arrives via `FederationReceiver`.
- **On 409 Conflict:** another actor claimed first. Return 409 to operator. Shadow updates from next CloudEvent.
- **On timeout / 5xx:** return 503 to operator — "owning service temporarily unreachable."
- **No background retry, no tentative states, no reconciliation.**

### Staleness Window

Between a failed proxy call and the arrival of the corrective CloudEvent, the shadow's displayed state may be stale. Duration is bounded by CloudEvent delivery latency (sub-second in Qhorus, seconds in webhook). The UI should communicate uncertainty.

## 10. Federation Receiver

`FederationReceiver` handles inbound CloudEvents from peer services:

1. Verify HMAC signature against the shared secret from the subscription
2. Extract `WorkItem` state snapshot from event data
3. Set `FederationSyncContext.activate()` (enables `FederationGuardStore` pass-through)
4. Upsert shadow WorkItem via `WorkItemStore.put()`
5. Clear `FederationSyncContext.deactivate()`
6. Record local audit entry with federation metadata
7. Trigger local SSE broadcast for connected clients

For `created` events: insert a new shadow WorkItem with `originServiceId` and `originWorkItemId` set.
For lifecycle events: update the existing shadow's status and fields from the snapshot.
For terminal events: update the shadow to terminal status. Remove lock-on tracking entry.

## 11. Audit Trail (D7)

### Owner Trail (Authoritative)

All mutations go through the owner's `WorkItemService`. The existing `AuditEntryStore` records every transition. Remote operations (claims/completions proxied from Service B) are recorded with the remote actor's identity — the owner's trail is complete.

### Shadow Trail (Local Activity Log)

The shadow service records proxied operations locally:

| Field | Value |
|-------|-------|
| `workItemId` | Shadow's local UUID |
| `event` | `FEDERATION_CLAIM_PROXIED`, `FEDERATION_COMPLETE_PROXIED`, etc. |
| `actor` | Local operator's identity |
| `detail` | JSON: `{ "originServiceId": "...", "originWorkItemId": "...", "remoteAuditEntryId": "..." }` |

The `remoteAuditEntryId` comes from the CloudEvent's `auditentryid` extension attribute, enabling cross-reference.

### Integrity

Hash chain integrity (casehub-work-ledger) is per-service only. Cross-service audit integrity is pointer-based — ProvenanceLink (#39) connects the chains into a unified PROV-O causal graph.

## 12. Security (D10)

### REST Commands (Proxy → Owner)

Authenticated via platform identity tokens (`casehub-platform-identity`). Standard bearer token flow — the proxy presents its service identity when calling the owner's REST API.

### CloudEvents (Owner → Consumer)

HMAC-signed with a shared secret established at subscription registration. The receiver verifies the signature before processing. This provides application-level payload integrity on top of transport-level TLS.

### Subscription Registration

Mutual authentication: both peers present platform identity tokens. The shared HMAC secret is exchanged during registration. Pairwise secrets in a mesh of N services means N*(N-1)/2 secrets — manageable at platform scale.

## 13. Multi-Tenancy (D11)

Federation subscriptions are tenant-scoped. `tenancyId` is mandatory on subscription registration. Cross-tenant federation is prohibited by default.

Shadow WorkItems inherit the owner's `tenancyId` and are subject to the same RLS policies as locally-created WorkItems. The `rls/` package in runtime operates on `WorkItem` records — since shadows ARE WorkItems (D4), RLS applies uniformly.

## 14. Lightweight Client Module

`casehub-work-client` provides:

- HTTP client for all WorkItem REST API operations (create, claim, complete, query, etc.)
- No JPA, no CDI, no Quarkus dependencies — plain Java + HTTP client
- Used by `FederationProxy` (D6)
- Usable standalone by any application that needs to interact with a remote casehub-work instance
- Addresses the gap identified in GE-20260805-a2aa1b (store SPI co-located with JPA)

## 15. Testing Strategy

| Level | What | How |
|-------|------|-----|
| Unit | `FederationGuardStore` decorator | `@QuarkusComponentTest` — verify rejects shadow mutations, passes local, passes with SyncContext |
| Unit | `FederationEventRouter` subscription matching | Pure Java — filter evaluation against WorkItem candidateGroups |
| Unit | `FederationReceiver` event processing | Mock store, verify upsert/audit calls |
| Unit | `FederationProxy` error handling | Mock HTTP client — test timeout, 409, 5xx paths |
| Integration | End-to-end federation | Two casehub-work instances in the same `@QuarkusTest` JVM, connected via in-memory webhook transport. Service A creates → Service B sees shadow → Service B claims → Service A records claim. |
| Integration | Subscription lifecycle | Register, create matching WorkItem, verify events delivered, create non-matching, verify not delivered |
| Integration | Guard store | Verify that ExpiryLifecycleService, WorkItemAssignmentService, etc. are blocked from mutating shadows |

## 16. Flyway Migration

| Table | Change |
|-------|--------|
| `work_item` | Add `origin_service_id VARCHAR(255)`, `origin_work_item_id UUID` (both nullable) |
| `federation_subscription` | New: `id UUID PK`, `peer_id VARCHAR`, `callback_url VARCHAR`, `tenancy_id VARCHAR NOT NULL`, `filter_json TEXT`, `capabilities_json TEXT`, `hmac_secret_hash VARCHAR`, `created_at TIMESTAMP`, `status VARCHAR` |
| `federation_subscription_tracking` | New: `subscription_id UUID FK`, `work_item_id UUID`, composite PK |
| `federation_audit_entry` | New: extends existing audit entry schema with `origin_service_id`, `origin_work_item_id`, `remote_audit_entry_id` |

Version range: per `docs/FLYWAY.md`, reserve a V-number block for the federation module.

## 17. Future Work

- **Qhorus FederationTransport** — deliver CloudEvents via Qhorus channels instead of webhooks. Qhorus provides built-in delivery guarantees, dead-letter handling, and speech act semantics.
- **A2A adapter** — `casehub-work-a2a` module mapping between A2A tasks and WorkItem lifecycle. Publishes Agent Cards for federation discovery.
- **#332 Coordinated rollback** — extends federation protocol with subtree-wide rollback messages.
- **#39 ProvenanceLink** — connects dual audit trails into unified PROV-O causal graph.
- **#97 Event mesh** — lifecycle events crossing boundaries via webhooks (now implementable with the federation infrastructure).
- **Cross-tenant federation** — optional opt-in for admin services viewing WorkItems across tenants.

## References

- [WS-HumanTask 1.1 (OASIS)](https://docs.oasis-open.org/bpel4people/ws-humantask-1.1-spec-cs-01.html) — Task Parent / Task Processor / Task Client separation, coordination protocol
- [A2A Protocol (Linux Foundation)](https://a2a-protocol.org/latest/specification/) — agent-to-agent task delegation, fluid roles, Agent Cards
- [CloudEvents (CNCF)](https://cloudevents.io/) — event envelope standard
- [CQRS Pattern (Azure)](https://learn.microsoft.com/en-us/azure/architecture/patterns/cqrs) — single-writer + read projections
- [Camunda 8.8 Architecture](https://camunda.com/blog/2025/11/whats-different-orchestration-cluster-camunda-88/) — industry consolidation to single-cluster
- [IBM EDA CQRS Patterns](https://ibm-cloud-architecture.github.io/refarch-eda/patterns/cqrs/) — event-driven projection patterns
- [Event-Driven.io Projections Guide](https://event-driven.io/en/projections_and_read_models_in_event_driven_architecture/) — read model design
- docs/LAYERING.md — casehub-work boundary rule
- docs/NORMATIVE-ALIGNMENT.md — WorkItem lifecycle to Qhorus speech act mapping
- docs/MODULES.md — module ownership and structural constraints
- ARC42STORIES.MD — primary architecture record
- GE-20260805-a2aa1b — WorkItemStore SPI co-location with JPA
- GE-20260629-45f4be — callerRef permanent reservation
- GE-20260521-87daa0 — @ObservesAsync in external JARs silently not dispatched
- GE-20260627-47c1eb — @RequestScoped fails in event handlers
- GE-20260421-cd3f95 — CDI observer recursive re-entry
