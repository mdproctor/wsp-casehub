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

### Feedback Loop Prevention

When `FederationReceiver` upserts a shadow and fires a `WorkItemLifecycleEvent` for SSE broadcast, the same CDI event chain that produces outbound CloudEvents is triggered. Without a guard, the shadow's lifecycle event would be routed back to the owner, creating an infinite cycle.

**Guard:** `FederationEventRouter` checks `event.workItem().originServiceId() != null` on every lifecycle event. Shadow-originated events (non-null `originServiceId`) are skipped — only locally-owned WorkItems are routed to subscribers. This single check in the router breaks all feedback loops regardless of transport or observer chain.

## 4. New Modules

| Module | Type | Purpose |
|--------|------|---------|
| `federation/` | Optional integration module | `FederationGuardStore`, `FederationProxy`, `FederationReceiver`, `FederationEventRouter`, subscription entities, REST endpoints |
| `client/` | Lightweight REST client | HTTP client for remote WorkItem operations. No JPA, no CDI. Used by `FederationProxy` and standalone consumers |

Both modules are activated by classpath presence, consistent with `engine-adapter/` and `qhorus/`. Non-federating deployments exclude them — zero overhead.

## 5. WorkItem Record Changes (D12)

Three nullable fields added to the `WorkItem` record in `api/`:

| Field | Type | Null means | Non-null means |
|-------|------|------------|----------------|
| `originServiceId` | `String` | Locally owned | Shadow — owned by this service |
| `originWorkItemId` | `UUID` | Locally owned | ID on the owning service |
| `originVersion` | `Long` | Locally owned (or first event not yet received) | Owner's `@Version` value at the time of the last applied federation event |

All three `WorkItemStore` backends (JPA, MongoDB, InMemory) persist these fields. The `WorkItem.Builder` gains three setter methods. Flyway migration adds three nullable columns to the `work_item` table.

`originVersion` is distinct from the shadow's own JPA `@Version` (which JPA manages independently and increments on every local `persistAndFlush()`). `WorkItemEntityMapper.copyFieldsToEntity()` deliberately excludes `version` from its field copy — JPA must control the OCC counter. `originVersion` is included in `copyFieldsToEntity()` and tracks the owner's mutation sequence for stale event detection (see §7 Event Ordering).

Consumer code can detect shadows: `if (workItem.originServiceId() != null) { /* shadow */ }`.

### Shadow Lookup

`WorkItemStore` gains a new method for shadow resolution by origin coordinates:

```
default Optional<WorkItem> findByOrigin(String originServiceId, UUID originWorkItemId) {
    return scanAll().stream()
        .filter(w -> originServiceId.equals(w.originServiceId())
                  && originWorkItemId.equals(w.originWorkItemId()))
        .findFirst();
}
```

The JPA backend overrides with an indexed query (`WHERE origin_service_id = ? AND origin_work_item_id = ?`). This follows the same pattern as `findByCallerRef()` — a default brute-force implementation with an efficient JPA override. A composite index on `(origin_service_id, origin_work_item_id)` is added in the Flyway migration (§16).

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

### Timer and Scheduler Interaction

`ExpiryLifecycleService`, `WorkItemAssignmentService`, `WorkItemTimerService`, and `WorkItemScheduleService` all call `WorkItemStore.put()`. For shadows, these calls hit `FederationGuardStore` and throw `FederatedWorkItemMutationException`.

**Prevention over exception handling:** rather than catching the exception in each service, shadow WorkItems are excluded at the query level:
- `WorkItemQuery.expired()` and `WorkItemQuery.claimExpired()` add `originServiceId IS NULL` to their predicates
- `FederationReceiver` does not schedule Quartz timers for shadows (it bypasses `WorkItemService`, which is where timers are scheduled)
- `WorkItemAssignmentService.assign()` is never called for shadows (assignment is managed by the owner)

The `FederationGuardStore` remains as a safety net — if a shadow mutation is attempted through any path not covered above, it throws rather than silently corrupting state.

## 7. Coordination Protocol (D3, D8)

### Dual Transport

| Channel | Carries | Transport | Why |
|---------|---------|-----------|-----|
| **Commands** | See command categorisation table below | REST (synchronous) | Operator needs confirmation that the command succeeded |
| **Events** | All `WorkEventType` values (26 types) — see event type table below | CloudEvents (asynchronous) | Fire-and-observe — projection updates, no confirmation needed |

### Command Categorisation

Every REST operation on a shadow WorkItem falls into one of three categories:

| Category | Operations | Rationale |
|----------|-----------|-----------|
| **Proxied** | claim, start, complete, reject, delegate, acceptDelegation, declineDelegation, release, suspend, resume, cancel, fault, obsolete, escalate, extend, updateDeadline, addLabel, removeLabel | Lifecycle mutations — must execute on the owner. Shadow updated when the resulting CloudEvent arrives. |
| **Shadow-local** | addNote, editNote, deleteNote, addLink, deleteLink, streamEvents (SSE), getById, listAll, inbox, clone | Read operations, consumer-local annotations stored as separate entities (WorkItemNote, WorkItemLink), and clone (creates a new local WorkItem with `originServiceId = null`, disconnected from federation — a valid use case for creating a local copy of a federated task). |
| **Blocked** | create (shadows are created by federation events only) | `FederationGuardStore` blocks create-via-put for items with `originServiceId != null`. |

### Event Types

All `WorkEventType` enum values are federation event types. The complete set:

| Event type | Status after | Notes |
|-----------|-------------|-------|
| `created` | PENDING | Initial creation |
| `assigned` | ASSIGNED | Auto-assignment or claim |
| `started` | IN_PROGRESS | Operator begins work |
| `completed` | COMPLETED | Terminal |
| `rejected` | REJECTED | Terminal |
| `faulted` | FAULTED | Terminal — system failure |
| `delegated` | DELEGATED | Forwarded to named actor |
| `delegation_accepted` | ASSIGNED | Delegatee accepted |
| `delegation_declined` | (previous) | Delegatee declined — reverts |
| `released` | PENDING | Claim released back to pool |
| `suspended` | SUSPENDED | Temporarily paused |
| `resumed` | IN_PROGRESS | Resumed from suspension |
| `cancelled` | CANCELLED | Terminal |
| `obsolete` | OBSOLETE | Terminal — context change |
| `expired` | EXPIRED | Terminal — deadline passed |
| `claim_expired` | (unchanged) | Claim deadline breached |
| `spawned` | (unchanged) | Child WorkItems created |
| `escalated` | ESCALATED | Terminal — policy exhausted |
| `deadline_extended` | (unchanged) | expiresAt moved forward |
| `sla_reassigned` | PENDING | Re-routed by breach policy |
| `sla_extended` | (unchanged) | Breach policy extended deadline |
| `signal_received` | (unchanged) | External signal routed |
| `manually_escalated` | PENDING | Actor escalated to new group |
| `progress_update` | (unchanged) | Progress reported |
| `label_added` | (unchanged) | Label attached |
| `label_removed` | (unchanged) | Label detached |

Since events carry full-state snapshots, every event brings the shadow fully current regardless of type. Completeness ensures shadows never remain stale due to unlisted event types.

### Event Envelope

All federation events are CloudEvents (CNCF graduated standard). Federation events use a distinct type prefix from local lifecycle events:

```
type:       io.casehub.work.federation.<event-type>
source:     urn:casehub:work:<service-id>
subject:    <work-item-id>
datacontenttype: application/json
Extensions:
  tenancyid:      <tenant-id>
  auditentryid:   <audit-entry-id on owner>
  workitemversion: <WorkItem @Version value>
```

**Why a separate prefix:** federation events carry a `WorkItem` state projection (subset of record fields), not a `WorkItemLifecycleEvent` (scalar event fields). The `io.casehub.work.federation.*` prefix distinguishes these structurally different payloads. It also prevents consumer-side CloudEvent observers (e.g., `WorkCloudEventInboundAdapter`) from accidentally processing federation events as local lifecycle events.

Event data is a **federation projection** of the `WorkItem` state (not a delta). The projection includes all operationally relevant fields and excludes owner-internal fields that are meaningless or sensitive across service boundaries:

**Excluded from federation projection:** `candidateScores`, `routingExperiences`, `excludedUsers`, `delegationChain`, `delegationDeclineTarget`, `accumulatedUnclaimedSeconds`, `lastReturnedToPoolAt`, `confidenceScore`.

Receivers overwrite the shadow's state from the projection. This is simpler than delta-based sync and tolerant of missed events — any single event brings the shadow fully current.

### Event Ordering

Each federation CloudEvent includes the owner's `@Version` value in the `workitemversion` extension attribute. The receiver stores this value in the shadow's `originVersion` field (§5) and uses it for stale event detection:

```
if (existingShadow.originVersion() != null
        && incomingVersion <= existingShadow.originVersion()) {
    // stale or duplicate — discard
    return;
}
shadow = shadow.toBuilder().originVersion(incomingVersion).build();
```

**Why `originVersion` is separate from `version`:** the shadow's JPA `@Version` field is managed by JPA independently — it increments on every local `persistAndFlush()`, counting shadow-side puts (1, 2, 3, ...). The owner's version counts owner-side mutations (which may be at 8 when the shadow is at 2). Comparing incoming owner versions against the shadow's JPA version produces incorrect ordering decisions. `originVersion` tracks the owner's sequence, `version` tracks the shadow's OCC sequence — they serve different purposes and must not be conflated.

### FederationEventRouter Hook Mechanism

`FederationEventRouter` uses `@ObservesAsync WorkItemLifecycleEvent` — the same async CDI channel that `WorkCloudEventAdapter` observes. This is the correct hook point because:

1. **Not `WorkItemObserver` SPI** — SPI observers run synchronously inside the emitter's transaction. HTTP calls to the transport would block the transaction.
2. **Not `@ObservesAsync CloudEvent`** — would require deserialising the CloudEvent payload to access WorkItem fields, and introduces a dependency on `WorkCloudEventAdapter` being active.
3. **`@ObservesAsync WorkItemLifecycleEvent`** — runs asynchronously, has direct access to the full `WorkItem` object via `event.workItem()`, and operates independently of the CloudEvent adapter.

**Originating-node constraint:** on distributed deployments using PostgreSQL LISTEN/NOTIFY broadcaster, wire-reconstructed events have `event.workItem() == null`. The router skips these — federation routing occurs only on the node where the mutation originated. This is correct: one routing per mutation is sufficient.

**Feedback loop guard:** the router checks `event.workItem().originServiceId() != null` and skips shadow-originated events (see §3 Feedback Loop Prevention).

**Audit entry cross-reference:** `WorkItemLifecycleEvent` will be extended with an optional `auditEntryId` field, populated by `WorkItemService.audit()` returning the persisted entry's ID. The router includes this in the CloudEvent's `auditentryid` extension attribute for cross-service audit trail linkage.

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
4. ALL subsequent lifecycle events for this WorkItem are delivered to Service B until terminal state (COMPLETED, REJECTED, CANCELLED, EXPIRED, OBSOLETE, FAULTED, ESCALATED)
5. No re-evaluation of the filter on subsequent events

Lock-on tracking is stored in a `federation_subscription_tracking` table: `(subscription_id, work_item_id)`.

### Catch-Up on Registration

1. **Activate the subscription first** — start lock-on tracking and event routing for the new subscription
2. **Then run the catch-up query** — find existing non-terminal WorkItems matching the filter and send a synthetic `created` event for each

This ordering prevents a TOCTOU race: a WorkItem created between the catch-up query and subscription activation would be missed. By activating first, the window is closed. Duplicates (WorkItem caught by both live routing and catch-up) are harmless — the receiver's upsert by `originWorkItemId` is idempotent, and version-based ordering (§7) discards stale events.

### Subscription Lifecycle Management

| Status | Meaning | Transition from |
|--------|---------|-----------------|
| `ACTIVE` | Events are delivered normally | Registration, manual reactivation |
| `SUSPENDED` | Delivery paused after repeated failures | `ACTIVE` (automatic after N consecutive failures) |
| `DEREGISTERED` | Peer unsubscribed; tracking entries retained for audit, cleaned up after 30 days | `ACTIVE`, `SUSPENDED` (via REST DELETE) |

**Deregistration:** `DELETE /federation/subscriptions/{id}`. In-flight events are best-effort (may or may not be delivered). Lock-on tracking entries are soft-deleted (status change) and cleaned up asynchronously.

**Health monitoring:** the `FederationTransport` increments `consecutive_failures` on delivery failure and resets to 0 on success. After 5 consecutive failures, the subscription is set to `SUSPENDED`. A suspended subscription emits no events. Manual reactivation via `PUT /federation/subscriptions/{id}/reactivate`.

**Subscription update:** peers update filter or capabilities via `PUT /federation/subscriptions/{id}`. Filter changes trigger a new catch-up for any WorkItems matching the new filter but not the old one.

### Filter Evaluation Algorithm

The subscription filter matches against `WorkItem.candidateGroups` (a comma-separated `String`) and `WorkItem.candidateUsers` (also comma-separated):

1. Split `WorkItem.candidateGroups` on `,` and strip whitespace from each element
2. Split `WorkItem.candidateUsers` on `,` and strip whitespace from each element
3. If the filter's `candidateGroups` array is non-empty: check whether **any** filter group matches **any** WorkItem group (set intersection, exact string match per group name)
4. If the filter's `candidateUsers` array is non-empty: check whether **any** filter user matches **any** WorkItem user (set intersection, exact string match)
5. candidateGroups and candidateUsers filters combine with **OR** — a match on either is sufficient
6. Empty filter arrays are ignored (an empty `candidateGroups` filter does not filter on groups)
7. If both filter arrays are empty, the subscription matches ALL WorkItems for the tenant

## 9. Federation Proxy (D6)

### Prerequisite: Extract WorkItemService Interface

`WorkItemService` is currently a concrete class. CDI `@Decorator` requires an interface — `implements ConcreteClass` is a compile error in Java. The existing `WorkItemStore` SPI is already an interface with three backends (JPA, MongoDB, InMemory); `WorkItemService` should follow the same pattern.

**Refactoring:** extract public lifecycle and query methods from `WorkItemService` into a `WorkItemService` interface in `api/`. The current class becomes `WorkItemServiceImpl implements WorkItemService` in `runtime/service/`. Injection sites update from the class to the interface. This is a mechanical refactoring with no behavioral change — it corrects an existing design asymmetry where `WorkItemStore` is a proper SPI but `WorkItemService` is not.

### FederationProxyService Decorator

`FederationProxyService` is a CDI `@Decorator` on the `WorkItemService` interface:

```
@Decorator @Priority(Interceptor.Priority.APPLICATION + 10)
class FederationProxyService implements WorkItemService {

    @Delegate WorkItemService delegate;
    @Inject FederationProxy proxy;

    WorkItem claim(UUID id, String claimantId) {
        WorkItem item = delegate.findById(id)
            .orElseThrow(() -> new WorkItemNotFoundException(id));
        if (item.originServiceId() != null) {
            return proxy.claim(item, claimantId);
        }
        return delegate.claim(id, claimantId);
    }
    // same pattern for all proxied lifecycle methods (§7 Command Categorisation)
    // query methods (findById, scan, etc.) delegate transparently
}
```

This provides a single interception point for all lifecycle operations. New operations added to `WorkItemService` are automatically covered (they delegate by default). REST resources require no modification. The extra `findById` in the decorator is a JPA L1 cache hit within the same persistence context — zero overhead.

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
2. Extract `WorkItem` state projection from event data
3. **Establish tenant context.** Extract `tenancyid` from the CloudEvent extension. Use `TenantContextRunner.runInTenantContext(tenancyId, ...)` to activate a CDI request scope for all subsequent steps (4–11). This is the same mechanism used by `WorkCloudEventInboundAdapter` for inbound CloudEvent processing. All JPA store methods are tenant-scoped via `CurrentPrincipal.tenancyId()` — without tenant context, `findByOrigin()` would include a null tenant filter and fail to find existing shadows, causing unbounded shadow duplication.
4. **Shadow lookup:** call `WorkItemStore.findByOrigin(originServiceId, originWorkItemId)` (§5) to find an existing shadow
5. **Version check:** if a shadow exists, compare incoming `workitemversion` against `shadow.originVersion()`. Discard if `incomingVersion <= shadow.originVersion()` (stale/duplicate). See §7 Event Ordering for why `originVersion` is used instead of the shadow's JPA `version`.
6. **callerRef namespacing:** prefix the owner's `callerRef` with `federation:<originServiceId>:` to prevent collisions with `WorkCloudEventInboundAdapter` (which uses `findByCallerRef(ce.getId())` for idempotency) and `QhorusWorkItemLifecycleAdapter` (which checks `QhorusCallerRef.isQhorus()` on terminal events)
7. Build shadow WorkItem with `originVersion` set from the incoming `workitemversion`
8. Set `FederationSyncContext.activate()` (enables `FederationGuardStore` pass-through)
9. Upsert shadow WorkItem via `WorkItemStore.put()`
10. Clear `FederationSyncContext.deactivate()`
11. Record local audit entry with federation metadata
12. Fire `WorkItemLifecycleEvent` via `WorkItemLifecycleEmitter.emit()` for SSE broadcast

Steps 4–12 execute within the `TenantContextRunner` scope established in step 3.

For `created` events: insert a new shadow WorkItem with `originServiceId`, `originWorkItemId`, and `originVersion` set.
For lifecycle events: update the existing shadow's status and fields from the projection.
For terminal events: update the shadow to terminal status. Remove lock-on tracking entry.

### Design: Bypassing WorkItemService

The receiver calls `WorkItemStore.put()` directly rather than going through `WorkItemService`. This is intentional — `WorkItemService` provides lifecycle validation, auto-assignment, timer management, label rule evaluation, spawn group enforcement, and other policies that must NOT execute on shadows:

| Concern | Why skipped for shadows |
|---------|----------------------|
| Lifecycle validation (status transition legality) | The owner already validated; the shadow accepts the owner's authoritative state |
| Auto-assignment (`WorkItemAssignmentService`) | Assignment is managed by the owner |
| Timer management (expiry, claim deadline) | Shadows don't have local timers — lifecycle timing is the owner's responsibility |
| Label rule engine | Owner-side rules; consumer may have different rules for its own WorkItems |
| Spawn group policy | Spawn groups are owner-local; shadows are independent projections |
| Exclusion policy | Owner enforces its own exclusion policy |

**What the receiver DOES replicate:**
- Audit entry creation (step 8) — local activity log with federation metadata
- SSE broadcast (step 9) — fires `WorkItemLifecycleEvent` via CDI, observed by `LocalWorkItemEventBroadcaster` for SSE and by `WorkCloudEventAdapter` for CloudEvent emission (the FederationEventRouter's originServiceId check prevents re-routing — see §3)

**Sync protocol:** when new concerns are added to `WorkItemService`, the federation integration test (§15) verifies that shadows reflect the correct state. The receiver's bypass list (above) is the explicit contract for what is and isn't replicated.

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

Federation audit entries use the existing `AuditEntry` entity and `audit_entry` table — no separate `federation_audit_entry` table. The federation metadata (`originServiceId`, `originWorkItemId`, `remoteAuditEntryId`) is stored in the `detail` JSON field, consistent with how other audit entry types store context-specific data. Cross-service audit correlation uses ProvenanceLink (#39), not dedicated columns.

### Integrity

Hash chain integrity (casehub-work-ledger) is per-service only. Cross-service audit integrity is pointer-based — ProvenanceLink (#39) connects the chains into a unified PROV-O causal graph.

## 12. Security (D10)

### REST Commands (Proxy → Owner)

Authenticated via platform identity tokens (`casehub-platform-identity`). Standard bearer token flow — the proxy presents its service identity when calling the owner's REST API.

### CloudEvents (Owner → Consumer)

HMAC-signed with a shared secret established at subscription registration. The receiver computes `HMAC-SHA256(secret, payload)` and compares against the signature in the CloudEvent. This provides application-level payload integrity on top of transport-level TLS.

**Secret storage:** the shared HMAC secret is stored **encrypted at rest** (not hashed) in the `federation_subscription` table (`hmac_secret_encrypted BYTEA`). HMAC verification requires the raw secret — `hash(secret)` cannot be used to compute `HMAC(secret, payload)`. Encryption uses platform credential management (`casehub-platform-credentials`), which provides JCE-backed encryption with key rotation support.

### Subscription Registration

Mutual authentication: both peers present platform identity tokens. The shared HMAC secret is exchanged during registration over the TLS-protected channel. The registering peer generates the secret and includes it in the registration payload; both sides store it encrypted. Pairwise secrets in a mesh of N services means N*(N-1)/2 secrets — manageable at platform scale.

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
| `work_item` | Add `origin_service_id VARCHAR(255)`, `origin_work_item_id UUID`, `origin_version BIGINT` (all nullable). Add composite index on `(origin_service_id, origin_work_item_id)` for `findByOrigin()` lookups. |
| `federation_subscription` | New: `id UUID PK`, `peer_id VARCHAR`, `callback_url VARCHAR`, `tenancy_id VARCHAR NOT NULL`, `filter_json TEXT`, `capabilities_json TEXT`, `hmac_secret_encrypted BYTEA`, `created_at TIMESTAMP`, `status VARCHAR`, `consecutive_failures INT DEFAULT 0`, `last_failure_at TIMESTAMP` |
| `federation_subscription_tracking` | New: `subscription_id UUID FK`, `work_item_id UUID`, composite PK |

Version range: per `docs/FLYWAY.md`, reserve a V-number block for the federation module.

## 17. Future Work

- **Qhorus FederationTransport** — deliver CloudEvents via Qhorus channels instead of webhooks. Qhorus provides built-in delivery guarantees, dead-letter handling, and speech act semantics.
- **A2A adapter** — `casehub-work-a2a` module mapping between A2A tasks and WorkItem lifecycle. Publishes Agent Cards for federation discovery.
- **#332 Coordinated rollback** — extends federation protocol with subtree-wide rollback messages.
- **#39 ProvenanceLink** — connects dual audit trails into unified PROV-O causal graph.
- **#97 Event mesh** — #97 is closed; federation infrastructure subsumes the webhook-based lifecycle event crossing that #97 originally envisioned. The subscription model (§8) and FederationTransport (§7) provide the concrete implementation path.
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
