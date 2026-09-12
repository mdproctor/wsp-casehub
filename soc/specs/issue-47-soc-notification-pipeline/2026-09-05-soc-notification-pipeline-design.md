# SOC Notification Pipeline Design

**Issue:** casehubio/soc#47
**Date:** 2026-09-05
**Branch:** issue-47-soc-notification-pipeline

---

## Overview

Wire SOC domain events into the platform notification subsystem so that analysts and SOC managers receive out-of-band alerts (Slack, email, webhook, in-app) when incidents are created, escalated, resolved, SLA-breached, or require containment approval. The platform provides the full subscription engine, dispatch pipeline, delivery channels, suppression, digest batching, and blocks-ui notification inbox — SOC provides event definitions, domain bridges, and default subscriptions.

## Architecture

### Platform Notification Pipeline (Existing)

```
┌──────────────────┐    ┌───────────────────┐    ┌─────────────────────┐
│  Domain Bridge    │───►│ Notification      │───►│ Notification        │
│  publishes        │    │ DataSource        │    │ SubscriptionEngine  │
│  SubscribableEvent│    │ (platform)        │    │ (filter matching)   │
└──────────────────┘    └───────────────────┘    └─────────┬───────────┘
                                                           │
                                                  SubscriptionMatched
                                                           │
                                                           ▼
                                               ┌─────────────────────┐
                                               │ NotificationDispatcher│
                                               │ 1. TargetResolver     │
                                               │ 2. SuppressionEvaluator│
                                               │ 3. TemplateResolver   │
                                               │ 4. ChannelRouter      │
                                               │ 5. Delivery + Digest  │
                                               └─────────────────────┘
```

### SOC Layer (This Epic)

```
┌─────────────────────────────────────────────────────┐
│  api/                                                │
│  ┌─────────────────────────────────────────────────┐ │
│  │ SocNotificationEvents                           │ │
│  │  - event type constants                         │ │
│  │  - EventFieldDescriptor declarations            │ │
│  │  - SubscribableEvent records (4 SOC-specific)   │ │
│  └─────────────────────────────────────────────────┘ │
│                                                      │
│  app/                                                │
│  ┌─────────────────────────────────────────────────┐ │
│  │ SocNotificationBridge                           │ │
│  │  - @Startup: register event types               │ │
│  │  - @ObservesAsync SocIncidentStatusChangedEvent  │ │
│  │  - @Observes(AFTER_SUCCESS) SlaBreachEvent      │ │
│  │  - @Observes(AFTER_SUCCESS) WorkItemLifecycleEvent│ │
│  │  - publishes to notification DataSource         │ │
│  └─────────────────────────────────────────────────┘ │
│  ┌─────────────────────────────────────────────────┐ │
│  │ SocNotificationSeeder                           │ │
│  │  - @Startup: seed SYSTEM subscriptions          │ │
│  │  - Idempotent upsert via content comparison     │ │
│  │  - Per-tenant seeding                           │ │
│  └─────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────┘
```

## Event Types

### SOC-Specific Events (Bridged by SocNotificationBridge)

| Event Type | CDI Source | Trigger Condition |
|---|---|---|
| `io.casehub.soc.incident.created` | `SocIncidentStatusChangedEvent` | `previousStatus == null` (new incident) |
| `io.casehub.soc.incident.escalated` | `SocIncidentStatusChangedEvent` | `newStatus == ESCALATED` |
| `io.casehub.soc.incident.resolved` | `SocIncidentStatusChangedEvent` | `newStatus == RESOLVED \|\| newStatus == FALSE_POSITIVE` |
| `io.casehub.soc.incident.sla_breached` | `SlaBreachEvent` | Any SLA breach on SOC incident work items |

Each event is a Java record implementing `SubscribableEvent` (provides `type()` and `tenancyId()`). The bridge enriches the lean domain event with case context (severity, assignedAnalystId, candidateGroups, tactic) by loading case state at dispatch time.

### Work Item Events (Bridged by SocNotificationBridge)

`WorkItemLifecycleEvent` has no `caseType` field — its filterable fields are `status`, `assigneeId`, `candidateGroups`, `outcome`, `types`. SOC cannot filter platform work item events to only incident-investigation cases. Therefore SOC bridges its own work item events by observing `WorkItemLifecycleEvent` and filtering by `callerRef` prefix (which encodes the case definition namespace).

| Event Type | CDI Source | Trigger Condition |
|---|---|---|
| `io.casehub.soc.workitem.created` | `WorkItemLifecycleEvent` | `status == PENDING` + callerRef matches SOC namespace |
| `io.casehub.soc.workitem.assigned` | `WorkItemLifecycleEvent` | `status == ASSIGNED` + callerRef matches SOC namespace |
| `io.casehub.soc.workitem.escalated` | `WorkItemLifecycleEvent` | `status == ESCALATED` + callerRef matches SOC namespace |
| `io.casehub.soc.workitem.completed` | `WorkItemLifecycleEvent` | `status == COMPLETED` + callerRef matches SOC namespace |

## Components

### api/ — SocNotificationEvents

Pure-Java constants and records. No CDI, no Quarkus dependencies.

**Event type constants:**
```java
public final class SocNotificationEvents {
    public static final String INCIDENT_CREATED = "io.casehub.soc.incident.created";
    public static final String INCIDENT_ESCALATED = "io.casehub.soc.incident.escalated";
    public static final String INCIDENT_RESOLVED = "io.casehub.soc.incident.resolved";
    public static final String INCIDENT_SLA_BREACHED = "io.casehub.soc.incident.sla_breached";
}
```

**SubscribableEvent records** (one per SOC event type):
```java
public record SocIncidentCreatedEvent(
    String tenancyId, String caseId, String severity,
    String tactic, String correlationKey, String actorId,
    Instant occurredAt
) implements SubscribableEvent {
    public String type() { return SocNotificationEvents.INCIDENT_CREATED; }
}
```

Similar records for escalated, resolved, SLA breached, and work item events — each carrying the fields needed for template resolution and filtering.

**System-generated events:** SOC notification events are system-generated (by the case engine, SLA timer, or ganglion). The `actorId` field uses a sentinel value `"system:soc-engine"` for events without a human actor. This satisfies `TemplateResolver`'s non-null `actorIdField` requirement and `NotificationSource`'s non-null `actorId` field.

**caseId type:** `SocIncidentStatusChangedEvent` carries `UUID caseId`. The notification event records use `String caseId` (via `UUID.toString()`) for template placeholder compatibility — `TemplateResolver.extractField()` calls `.toString()` on extracted values.

**EventFieldDescriptor declarations** (shared across event type registrations):
- `severity` (string) — P1/P2/P3/P4 for filtering
- `tactic` (string) — MITRE ATT&CK tactic for filtering
- `caseId` (string) — incident identifier for actionUrl
- `correlationKey` (string) — affected entity
- `assignedAnalystId` (string) — for EVENT_FIELD targeting
- `candidateGroups` (string) — for EVENT_FIELD targeting

### app/ — SocNotificationBridge

Single `@ApplicationScoped` CDI bean. Three methods:

**`onStartup(@Observes StartupEvent)`** — registers four SOC event types with `EventTypeRegistry`:
```java
eventTypeRegistry.register(new EventTypeDescriptor(
    SocNotificationEvents.INCIDENT_CREATED,
    "Incident Created",
    "New SOC incident enters triage",
    List.of(
        new EventFieldDescriptor("severity", "Severity", "string"),
        new EventFieldDescriptor("tactic", "ATT&CK Tactic", "string"),
        new EventFieldDescriptor("correlationKey", "Affected Entity", "string")
    )
));
```

**`onIncidentStatusChanged(@ObservesAsync SocIncidentStatusChangedEvent)`** — dispatches incident created, escalated, and resolved events. Loads case context for enrichment. Publishes to notification DataSource following the `WorkItemSubscriptionBridge` pattern:
```java
dataSourceRegistryInstance.get()
    .resolveSource(NOTIFICATION_DATASOURCE_PATH, "platform")
    .ifPresent(ds -> ds.add(new SocIncidentCreatedEvent(
        event.tenancyId(), event.caseId().toString(), severity,
        tactic, correlationKey, "system:soc-engine",
        event.occurredAt())));
```

Uses `@ObservesAsync` because `SocIncidentStatusObserver` fires via `Event.fireAsync()`.

**Escalation dedup:** When `newStatus == ESCALATED` and the escalation was caused by an SLA breach (detectable via preceding `SlaBreachEvent` for the same caseId), only `incident.sla_breached` fires — `incident.escalated` is suppressed to avoid double-notification.

**`onSlaBreach(@Observes(during = AFTER_SUCCESS) SlaBreachEvent)`** — dispatches SLA breached events for SOC incident work items only. Filters by `callerRef` prefix matching SOC case definition namespace. Enriches from `SlaBreachContext.task()` (carries `taskId`, `callerRef`, `title`, `candidateGroups`) — resolves `callerRef` → caseId → case context to extract severity and tactic. Uses `@Observes(during = AFTER_SUCCESS)` because SLA breach fires synchronously within the timer transaction.

**`onWorkItemEvent(@Observes(during = AFTER_SUCCESS) WorkItemLifecycleEvent)`** — dispatches SOC-specific work item events. Filters by `callerRef` prefix to only process SOC incident work items. Publishes `io.casehub.soc.workitem.*` events.

**Graceful degradation:** Uses `Instance<DataSourceRegistry>` and `Instance<EventTypeRegistry>` with `isUnsatisfied()` guards on both. If either SPI is unavailable, logs a warning and returns — no crash, no notification.

**Restart resilience:** `SocIncidentStatusObserver` tracks per-case status in a `ConcurrentHashMap` that is lost on restart. After restart, the first status update for in-flight cases will have `previousStatus == null`, which maps to `incident.created`. This generates spurious "new incident" notifications for pre-existing incidents. Accepted risk for now — persistent status tracking is a future enhancement. The impact is bounded: only in-flight incidents at restart time are affected, and mute rules can suppress specific entities.

### app/ — SocNotificationSeeder

`@ApplicationScoped` CDI bean. Seeds SYSTEM subscriptions at startup.

**`onStartup(@Observes StartupEvent)`** — for each known tenant:
1. Query existing SYSTEM subscriptions via `SubscriptionStore.find()`
2. For each of the 8 event types, check if a matching SYSTEM subscription exists
3. If missing: create via `SubscriptionStore.store()`
4. If exists but template differs: update via `SubscriptionStore.update()`

**Tenant discovery:** At startup, queries for existing SYSTEM subscriptions per known tenant. For the initial implementation, uses a configured tenant list from `application.properties` (`casehub.soc.notification.tenants`). Multi-tenant auto-discovery is a platform capability not yet available — the configured list is a pragmatic starting point.

**Idempotency:** `SubscriptionQuery` cannot filter by `eventType` — only by `ownerId`, `tenancyId`, `scope`, `enabled`. The seeder loads all SYSTEM subscriptions for the tenant, then filters in-memory by `eventType`. For 8 event types this is efficient. Content comparison on template fields (titlePattern, bodyPattern, severity, category, actionUrlPattern). No version field needed — if the template content matches, no update occurs.

## Default Subscription Matrix

### SOC-Specific Event Subscriptions

| Event | Target | Severity | Category |
|---|---|---|---|
| `incident.created` (P1/P2) | GROUP: soc-manager, soc-on-call | URGENT | soc-incident |
| `incident.created` (P3/P4) | GROUP: soc-tier1-analyst | INFO | soc-incident |
| `incident.escalated` | GROUP: soc-manager | URGENT | soc-incident |
| `incident.resolved` | ENTITY_WATCHERS: incident | INFO | soc-incident |
| `incident.sla_breached` | EVENT_FIELD: assignedAnalystId + GROUP: soc-manager | URGENT | soc-sla |

### SOC Work Item Event Subscriptions

| Event | Target | Severity | Category |
|---|---|---|---|
| `soc.workitem.created` (containment approval) | EVENT_FIELD: candidateGroups | URGENT | soc-containment |
| `soc.workitem.assigned` | EVENT_FIELD: assigneeId | WARNING | soc-workitem |
| `soc.workitem.escalated` | EVENT_FIELD: candidateGroups | URGENT | soc-workitem |
| `soc.workitem.completed` | EVENT_FIELD: assigneeId | INFO | soc-workitem |

## Notification Templates

Templates use `{field}` placeholder interpolation via `TemplateResolver`. Each template must declare `entityType`, `entityIdField`, and `actorIdField` — the resolver returns null (skips notification) if either entityId or actorId resolves to null on the POJO. MethodHandle-based field extraction is cached.

Examples:

**Incident Created (P1/P2):**
- Title: `URGENT: {severity} incident — {correlationKey}`
- Body: `New incident created. Tactic: {tactic}. Requires immediate triage.`
- ActionUrl: `/soc/incidents/{caseId}`

**Containment Approval Needed:**
- Title: `Containment approval required — {description}`
- Body: `Action: {actionType}. Risk score: {riskScore}. Awaiting approval.`
- ActionUrl: `/soc/workitems/{workItemId}`

**SLA Breached:**
- Title: `SLA BREACH: {severity} incident {caseId}`
- Body: `Response deadline exceeded. Escalation in progress.`
- ActionUrl: `/soc/incidents/{caseId}`

## Filter Strategy

SYSTEM subscriptions are unfiltered — every significant SOC event generates a notification for compliance. Users may create additional USER-scope subscriptions with filters:

- By severity: `severity == 'P1'`
- By tactic: `tactic == 'credential-access'`
- By assignment: `assignedAnalystId == $me`

Filter fields are declared via `EventFieldDescriptor` at event type registration. The subscription editor in blocks-ui renders available fields and operators for user-created subscriptions.

## UI Integration

No SOC-specific notification UI is needed. The blocks-ui `notification-inbox` component suite provides:

| Component | Embedding |
|---|---|
| `notification-bell` | SOC app header — unread count badge |
| `notification-inbox` | SOC dashboard panel — full inbox |
| `notification-preferences` | SOC user settings — channel preferences, quiet hours |
| `subscription-editor` | SOC user settings — create custom subscriptions |
| `mute-list` | SOC user settings — manage mute rules |
| `snooze-control` | SOC app header — activate/cancel snooze |

SOC pages embeds these components with `baseUrl` pointing to the platform notification REST endpoints. The event type picker in `subscription-editor` discovers SOC event types via the `/subscriptions/event-types` endpoint (backed by `EventTypeRegistry`).

## Testing Strategy

### Unit Tests
- `SocNotificationBridge`: verify correct event type dispatched for each status transition; verify enrichment loads case context; verify graceful degradation when DataSource unavailable
- `SocNotificationSeeder`: verify idempotent creation; verify content-comparison update; verify per-tenant seeding
- `SocIncidentCreatedEvent` (and siblings): verify `SubscribableEvent` contract (type(), tenancyId())

### Integration Tests
- End-to-end: fire `SocIncidentStatusChangedEvent` → verify `NotificationStore` contains the notification
- SYSTEM subscription matching: seed subscription → fire event → verify `SubscriptionMatched` CDI event
- Suppression: seed subscription → add mute rule → fire event → verify no notification stored

## Dependencies

Already on classpath:
- `casehub-platform-notifications` (app/pom.xml line 172) — REST endpoints, subscription engine, dispatch pipeline

No new Maven dependencies required.

## References

- `SubscriptionEngine.java` — platform subscription matching engine (DataSource-based)
- `NotificationDispatcher.java` — platform dispatch pipeline (target → suppression → template → channel → deliver)
- `WorkItemSubscriptionBridge` — platform bridge pattern (precedent for SOC bridge)
- `SocIncidentStatusChangedEvent` — existing CDI event for status transitions
- `SocIncidentStatusObserver` — fires status change events via `fireAsync()`
- `SocContainmentCommitmentBridge` — existing single-class CDI observer pattern in SOC
- `blocks-ui/components/notification-inbox/` — generic notification UI components
- `EventTypeRegistry`, `EventTypeDescriptor`, `EventFieldDescriptor` — platform event type SPI
- `SubscribableEvent` — platform interface for notification DataSource events
- `NotificationStore`, `NotificationInput`, `Notification` — platform notification persistence SPI
- `SubscriptionStore`, `Subscription`, `SubscriptionInput` — platform subscription persistence SPI
- `SuppressionStore`, `MuteRule`, `Snooze` — platform suppression SPI
- DORA Article 17 — mandatory incident detection and escalation timelines
- SOC2 CC7.3 — demonstrable notification chains for critical incidents
