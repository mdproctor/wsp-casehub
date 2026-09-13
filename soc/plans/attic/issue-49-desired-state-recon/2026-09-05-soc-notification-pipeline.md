# SOC Notification Pipeline Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #47 — Epic: SOC notification pipeline — analyst alerting, digest batching, suppression
**Issue group:** #47

**Goal:** Wire SOC domain events into the platform notification subsystem so analysts and SOC managers receive out-of-band alerts when incidents are created, escalated, resolved, SLA-breached, or require containment approval.

**Architecture:** SOC defines eight SubscribableEvent types (four incident lifecycle, four work item) published to the platform notification DataSource by a single CDI bridge class. A startup seeder creates SYSTEM-scope subscriptions with notification templates. The platform handles subscription matching, template resolution, suppression, digest batching, and delivery.

**Tech Stack:** Java 21, Quarkus 3.32.2, CDI events, platform notification subsystem (SubscribableEvent, DataSourceRegistry, SubscriptionStore, EventTypeRegistry)

## Global Constraints

- `api/` module: pure Java — no CDI, no Quarkus, no JPA annotations
- `app/` module: CDI + Quarkus
- Follow `WorkItemSubscriptionBridge` pattern for DataSource publishing: `Instance<>` guards, `"platform"` tenancy, `Optional` handling
- System-generated events use sentinel actorId `"system:soc-engine"`
- Event type naming: `io.casehub.soc.incident.<type>` and `io.casehub.soc.workitem.<type>`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
- Scoped test: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl <module> -am test -Dtest=<TestClass> -Dsurefire.failIfNoSpecifiedTests=false`
- All commits reference `Refs #47`

---

## Batch 1: Foundation — event type constants and notification records

### Task 1: Notification event types and SubscribableEvent records

**Files:**
- Create: `api/src/main/java/io/casehub/soc/domain/SocNotificationEvents.java`
- Create: `api/src/main/java/io/casehub/soc/domain/notification/SocIncidentNotification.java`
- Create: `api/src/main/java/io/casehub/soc/domain/notification/SocWorkItemNotification.java`
- Test: `api/src/test/java/io/casehub/soc/domain/notification/SocIncidentNotificationTest.java`
- Test: `api/src/test/java/io/casehub/soc/domain/notification/SocWorkItemNotificationTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.subscription.SubscribableEvent` (from casehub-platform-api dependency)
- Consumes: `io.casehub.platform.api.subscription.EventTypeDescriptor`, `EventFieldDescriptor`
- Produces: `SocNotificationEvents` constants (used by Tasks 2, 3), `SocIncidentNotification` record (used by Task 2), `SocWorkItemNotification` record (used by Task 2)

- [ ] **Step 1: Write failing test for SocIncidentNotification**

```java
package io.casehub.soc.domain.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;
import io.casehub.soc.domain.SocNotificationEvents;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.Arguments;
import org.junit.jupiter.params.provider.MethodSource;

import java.time.Instant;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.*;

class SocIncidentNotificationTest {

    @Test
    void implementsSubscribableEvent() {
        var event = new SocIncidentNotification(
            SocNotificationEvents.INCIDENT_CREATED,
            "tenant-1", "case-uuid", "P1",
            "credential-access", "server-042",
            "system:soc-engine", "analyst-1", Instant.now());
        assertInstanceOf(SubscribableEvent.class, event);
    }

    @Test
    void typeReturnsEventType() {
        var event = new SocIncidentNotification(
            SocNotificationEvents.INCIDENT_ESCALATED,
            "tenant-1", "case-uuid", "P1",
            "lateral-movement", "server-042",
            "system:soc-engine", null, Instant.now());
        assertEquals("io.casehub.soc.incident.escalated", event.type());
    }

    @Test
    void tenancyIdReturnsTenantId() {
        var event = new SocIncidentNotification(
            SocNotificationEvents.INCIDENT_CREATED,
            "my-tenant", "case-uuid", "P2",
            null, "host-1",
            "system:soc-engine", null, Instant.now());
        assertEquals("my-tenant", event.tenancyId());
    }

    @ParameterizedTest
    @MethodSource("allIncidentEventTypes")
    void allEventTypesAreValid(String eventType) {
        var event = new SocIncidentNotification(
            eventType, "t", "c", "P1", null, "h",
            "system:soc-engine", null, Instant.now());
        assertEquals(eventType, event.type());
    }

    static Stream<Arguments> allIncidentEventTypes() {
        return Stream.of(
            Arguments.of(SocNotificationEvents.INCIDENT_CREATED),
            Arguments.of(SocNotificationEvents.INCIDENT_ESCALATED),
            Arguments.of(SocNotificationEvents.INCIDENT_RESOLVED),
            Arguments.of(SocNotificationEvents.INCIDENT_SLA_BREACHED));
    }

    @Test
    void rejectsNullEventType() {
        assertThrows(NullPointerException.class, () ->
            new SocIncidentNotification(
                null, "t", "c", "P1", null, "h",
                "system:soc-engine", null, Instant.now()));
    }

    @Test
    void rejectsNullTenancyId() {
        assertThrows(NullPointerException.class, () ->
            new SocIncidentNotification(
                SocNotificationEvents.INCIDENT_CREATED,
                null, "c", "P1", null, "h",
                "system:soc-engine", null, Instant.now()));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl api -am test -Dtest=SocIncidentNotificationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes not found

- [ ] **Step 3: Create SocNotificationEvents constants**

```java
package io.casehub.soc.domain;

import io.casehub.platform.api.subscription.EventFieldDescriptor;
import io.casehub.platform.api.subscription.EventTypeDescriptor;

import java.util.List;

public final class SocNotificationEvents {

    private SocNotificationEvents() {}

    public static final String INCIDENT_CREATED = "io.casehub.soc.incident.created";
    public static final String INCIDENT_ESCALATED = "io.casehub.soc.incident.escalated";
    public static final String INCIDENT_RESOLVED = "io.casehub.soc.incident.resolved";
    public static final String INCIDENT_SLA_BREACHED = "io.casehub.soc.incident.sla_breached";

    public static final String WORKITEM_CREATED = "io.casehub.soc.workitem.created";
    public static final String WORKITEM_ASSIGNED = "io.casehub.soc.workitem.assigned";
    public static final String WORKITEM_ESCALATED = "io.casehub.soc.workitem.escalated";
    public static final String WORKITEM_COMPLETED = "io.casehub.soc.workitem.completed";

    public static final String ACTOR_SYSTEM = "system:soc-engine";

    private static final List<EventFieldDescriptor> INCIDENT_FIELDS = List.of(
        new EventFieldDescriptor("severity", "Severity", "string"),
        new EventFieldDescriptor("tactic", "ATT&CK Tactic", "string"),
        new EventFieldDescriptor("caseId", "Incident ID", "string"),
        new EventFieldDescriptor("correlationKey", "Affected Entity", "string"),
        new EventFieldDescriptor("assignedAnalystId", "Assigned Analyst", "string"));

    private static final List<EventFieldDescriptor> WORKITEM_FIELDS = List.of(
        new EventFieldDescriptor("workItemId", "Work Item ID", "string"),
        new EventFieldDescriptor("assigneeId", "Assignee", "string"),
        new EventFieldDescriptor("candidateGroups", "Candidate Groups", "string"),
        new EventFieldDescriptor("title", "Title", "string"),
        new EventFieldDescriptor("caseId", "Incident ID", "string"));

    public static List<EventTypeDescriptor> incidentDescriptors() {
        return List.of(
            new EventTypeDescriptor(INCIDENT_CREATED, "Incident Created",
                "New SOC incident enters triage", INCIDENT_FIELDS),
            new EventTypeDescriptor(INCIDENT_ESCALATED, "Incident Escalated",
                "SOC incident escalated to higher tier", INCIDENT_FIELDS),
            new EventTypeDescriptor(INCIDENT_RESOLVED, "Incident Resolved",
                "SOC incident resolved or classified as false positive", INCIDENT_FIELDS),
            new EventTypeDescriptor(INCIDENT_SLA_BREACHED, "SLA Breached",
                "SOC incident work item SLA breached", INCIDENT_FIELDS));
    }

    public static List<EventTypeDescriptor> workItemDescriptors() {
        return List.of(
            new EventTypeDescriptor(WORKITEM_CREATED, "SOC Work Item Created",
                "SOC investigation work item created", WORKITEM_FIELDS),
            new EventTypeDescriptor(WORKITEM_ASSIGNED, "SOC Work Item Assigned",
                "SOC investigation work item assigned to analyst", WORKITEM_FIELDS),
            new EventTypeDescriptor(WORKITEM_ESCALATED, "SOC Work Item Escalated",
                "SOC investigation work item escalated", WORKITEM_FIELDS),
            new EventTypeDescriptor(WORKITEM_COMPLETED, "SOC Work Item Completed",
                "SOC investigation work item completed", WORKITEM_FIELDS));
    }
}
```

- [ ] **Step 4: Create SocIncidentNotification record**

```java
package io.casehub.soc.domain.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

import java.time.Instant;
import java.util.Objects;

public record SocIncidentNotification(
    String eventType,
    String tenancyId,
    String caseId,
    String severity,
    String tactic,
    String correlationKey,
    String actorId,
    String assignedAnalystId,
    Instant occurredAt
) implements SubscribableEvent {

    public SocIncidentNotification {
        Objects.requireNonNull(eventType, "eventType");
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(caseId, "caseId");
        Objects.requireNonNull(actorId, "actorId");
        Objects.requireNonNull(occurredAt, "occurredAt");
    }

    @Override
    public String type() { return eventType; }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl api -am test -Dtest=SocIncidentNotificationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 6: Write failing test for SocWorkItemNotification**

```java
package io.casehub.soc.domain.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;
import io.casehub.soc.domain.SocNotificationEvents;
import org.junit.jupiter.api.Test;

import java.time.Instant;

import static org.junit.jupiter.api.Assertions.*;

class SocWorkItemNotificationTest {

    @Test
    void implementsSubscribableEvent() {
        var event = new SocWorkItemNotification(
            SocNotificationEvents.WORKITEM_CREATED,
            "tenant-1", "wi-uuid", "case-uuid",
            "Containment: isolate host", "analyst-1",
            "soc-manager", "system:soc-engine", Instant.now());
        assertInstanceOf(SubscribableEvent.class, event);
    }

    @Test
    void typeReturnsEventType() {
        var event = new SocWorkItemNotification(
            SocNotificationEvents.WORKITEM_ASSIGNED,
            "tenant-1", "wi-uuid", "case-uuid",
            "Review triage", "analyst-2",
            "soc-tier1-analyst", "system:soc-engine", Instant.now());
        assertEquals("io.casehub.soc.workitem.assigned", event.type());
    }

    @Test
    void rejectsNullEventType() {
        assertThrows(NullPointerException.class, () ->
            new SocWorkItemNotification(
                null, "t", "wi", "c", "title",
                null, null, "system:soc-engine", Instant.now()));
    }
}
```

- [ ] **Step 7: Create SocWorkItemNotification record**

```java
package io.casehub.soc.domain.notification;

import io.casehub.platform.api.subscription.SubscribableEvent;

import java.time.Instant;
import java.util.Objects;

public record SocWorkItemNotification(
    String eventType,
    String tenancyId,
    String workItemId,
    String caseId,
    String title,
    String assigneeId,
    String candidateGroups,
    String actorId,
    Instant occurredAt
) implements SubscribableEvent {

    public SocWorkItemNotification {
        Objects.requireNonNull(eventType, "eventType");
        Objects.requireNonNull(tenancyId, "tenancyId");
        Objects.requireNonNull(workItemId, "workItemId");
        Objects.requireNonNull(actorId, "actorId");
        Objects.requireNonNull(occurredAt, "occurredAt");
    }

    @Override
    public String type() { return eventType; }
}
```

- [ ] **Step 8: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl api -am test -Dtest=SocIncidentNotificationTest,SocWorkItemNotificationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add api/src/main/java/io/casehub/soc/domain/SocNotificationEvents.java \
  api/src/main/java/io/casehub/soc/domain/notification/SocIncidentNotification.java \
  api/src/main/java/io/casehub/soc/domain/notification/SocWorkItemNotification.java \
  api/src/test/java/io/casehub/soc/domain/notification/SocIncidentNotificationTest.java \
  api/src/test/java/io/casehub/soc/domain/notification/SocWorkItemNotificationTest.java
git commit -m "feat(#47): add SOC notification event types and SubscribableEvent records

Defines 8 event type constants (4 incident lifecycle, 4 work item)
and two parameterised SubscribableEvent records for the notification
DataSource.

Refs #47"
```

---

## Batch 2: Bridge and seeder

### Task 2: SocNotificationBridge — CDI observers and DataSource publishing

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/notification/SocNotificationBridge.java`
- Test: `app/src/test/java/io/casehub/soc/engine/notification/SocNotificationBridgeTest.java`

**Interfaces:**
- Consumes: `SocNotificationEvents` (constants, descriptors), `SocIncidentNotification`, `SocWorkItemNotification` (from Task 1)
- Consumes: `SocIncidentStatusChangedEvent` (existing CDI event), `SlaBreachEvent` (from casehub-work), `WorkItemLifecycleEvent` (from casehub-work-api)
- Consumes: `Instance<DataSourceRegistry>`, `Instance<EventTypeRegistry>` (platform SPIs)
- Consumes: `CaseInstanceRepository` (for case context enrichment)
- Produces: Notification events published to platform DataSource (consumed by SubscriptionEngine)

- [ ] **Step 1: Write failing test for incident status → notification dispatch**

```java
package io.casehub.soc.engine.notification;

import io.casehub.platform.api.datasource.DataSource;
import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.platform.api.subscription.EventTypeDescriptor;
import io.casehub.platform.api.subscription.EventTypeRegistry;
import io.casehub.platform.api.subscription.SubscribableEvent;
import io.casehub.soc.domain.SocIncidentStatusChangedEvent;
import io.casehub.soc.domain.SocNotificationEvents;
import io.casehub.soc.domain.notification.SocIncidentNotification;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.UUID;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.ArgumentMatchers.eq;
import static org.mockito.Mockito.*;

class SocNotificationBridgeTest {

    private SocNotificationBridge bridge;
    private DataSourceRegistry dataSourceRegistry;
    private EventTypeRegistry eventTypeRegistry;
    private DataSource<Object> dataSource;
    private List<Object> publishedEvents;

    @BeforeEach
    @SuppressWarnings("unchecked")
    void setUp() {
        dataSourceRegistry = mock(DataSourceRegistry.class);
        eventTypeRegistry = mock(EventTypeRegistry.class);
        dataSource = mock(DataSource.class);
        publishedEvents = new ArrayList<>();

        when(dataSourceRegistry.resolveSource(any(), eq("platform")))
            .thenReturn(Optional.of(dataSource));
        doAnswer(inv -> { publishedEvents.add(inv.getArgument(0)); return null; })
            .when(dataSource).add(any());

        Instance<DataSourceRegistry> dsInstance = mock(Instance.class);
        when(dsInstance.isUnsatisfied()).thenReturn(false);
        when(dsInstance.get()).thenReturn(dataSourceRegistry);

        Instance<EventTypeRegistry> etInstance = mock(Instance.class);
        when(etInstance.isUnsatisfied()).thenReturn(false);
        when(etInstance.get()).thenReturn(eventTypeRegistry);

        bridge = new SocNotificationBridge(dsInstance, etInstance);
    }

    @Test
    void newIncidentPublishesCreatedEvent() {
        var event = new SocIncidentStatusChangedEvent(
            UUID.randomUUID(), "tenant-1", null, "TRIAGING", Instant.now());
        bridge.onIncidentStatusChanged(event);

        assertEquals(1, publishedEvents.size());
        var notification = (SocIncidentNotification) publishedEvents.get(0);
        assertEquals(SocNotificationEvents.INCIDENT_CREATED, notification.type());
        assertEquals("tenant-1", notification.tenancyId());
        assertEquals(SocNotificationEvents.ACTOR_SYSTEM, notification.actorId());
    }

    @Test
    void escalatedIncidentPublishesEscalatedEvent() {
        var event = new SocIncidentStatusChangedEvent(
            UUID.randomUUID(), "tenant-1", "INVESTIGATING", "ESCALATED", Instant.now());
        bridge.onIncidentStatusChanged(event);

        assertEquals(1, publishedEvents.size());
        var notification = (SocIncidentNotification) publishedEvents.get(0);
        assertEquals(SocNotificationEvents.INCIDENT_ESCALATED, notification.type());
    }

    @Test
    void resolvedIncidentPublishesResolvedEvent() {
        var event = new SocIncidentStatusChangedEvent(
            UUID.randomUUID(), "tenant-1", "CONTAINING", "RESOLVED", Instant.now());
        bridge.onIncidentStatusChanged(event);

        assertEquals(1, publishedEvents.size());
        var notification = (SocIncidentNotification) publishedEvents.get(0);
        assertEquals(SocNotificationEvents.INCIDENT_RESOLVED, notification.type());
    }

    @Test
    void falsePositivePublishesResolvedEvent() {
        var event = new SocIncidentStatusChangedEvent(
            UUID.randomUUID(), "tenant-1", "TRIAGING", "FALSE_POSITIVE", Instant.now());
        bridge.onIncidentStatusChanged(event);

        assertEquals(1, publishedEvents.size());
        var notification = (SocIncidentNotification) publishedEvents.get(0);
        assertEquals(SocNotificationEvents.INCIDENT_RESOLVED, notification.type());
    }

    @Test
    void intermediateTransitionIsIgnored() {
        var event = new SocIncidentStatusChangedEvent(
            UUID.randomUUID(), "tenant-1", "TRIAGING", "INVESTIGATING", Instant.now());
        bridge.onIncidentStatusChanged(event);

        assertTrue(publishedEvents.isEmpty());
    }

    @Test
    void gracefulDegradationWhenDataSourceUnavailable() {
        Instance<DataSourceRegistry> dsInstance = mock(Instance.class);
        when(dsInstance.isUnsatisfied()).thenReturn(true);
        Instance<EventTypeRegistry> etInstance = mock(Instance.class);
        when(etInstance.isUnsatisfied()).thenReturn(true);

        bridge = new SocNotificationBridge(dsInstance, etInstance);
        var event = new SocIncidentStatusChangedEvent(
            UUID.randomUUID(), "tenant-1", null, "TRIAGING", Instant.now());

        assertDoesNotThrow(() -> bridge.onIncidentStatusChanged(event));
        assertTrue(publishedEvents.isEmpty());
    }

    @Test
    void registersAllEventTypesOnStartup() {
        bridge.onStartup(null);

        verify(eventTypeRegistry, times(8)).register(any(EventTypeDescriptor.class));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl app -am test -Dtest=SocNotificationBridgeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — SocNotificationBridge not found

- [ ] **Step 3: Implement SocNotificationBridge**

```java
package io.casehub.soc.engine.notification;

import io.casehub.platform.api.datasource.DataSourceRegistry;
import io.casehub.platform.api.subscription.EventTypeRegistry;
import io.casehub.soc.domain.SocCaseTypes;
import io.casehub.soc.domain.SocNotificationEvents;
import io.casehub.soc.domain.SocIncidentStatusChangedEvent;
import io.casehub.soc.domain.notification.SocIncidentNotification;
import io.casehub.soc.domain.notification.SocWorkItemNotification;
import io.casehub.work.api.WorkItemLifecycleEvent;
import io.casehub.work.runtime.event.SlaBreachEvent;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Instant;

import static io.casehub.platform.api.subscription.SubscriptionConstants.NOTIFICATION_DATASOURCE_PATH;

@ApplicationScoped
public class SocNotificationBridge {

    private static final Logger LOG = Logger.getLogger(SocNotificationBridge.class);
    private static final String SOC_CALLER_REF_PREFIX = SocCaseTypes.INCIDENT_INVESTIGATION + ":";

    private final Instance<DataSourceRegistry> dataSourceRegistryInstance;
    private final Instance<EventTypeRegistry> eventTypeRegistryInstance;

    @Inject
    public SocNotificationBridge(Instance<DataSourceRegistry> dataSourceRegistryInstance,
                                 Instance<EventTypeRegistry> eventTypeRegistryInstance) {
        this.dataSourceRegistryInstance = dataSourceRegistryInstance;
        this.eventTypeRegistryInstance = eventTypeRegistryInstance;
    }

    void onStartup(@Observes StartupEvent event) {
        if (eventTypeRegistryInstance.isUnsatisfied()) {
            LOG.warn("EventTypeRegistry unavailable — SOC notification event types not registered");
            return;
        }
        var registry = eventTypeRegistryInstance.get();
        SocNotificationEvents.incidentDescriptors().forEach(registry::register);
        SocNotificationEvents.workItemDescriptors().forEach(registry::register);
        LOG.info("SOC notification event types registered (8 types)");
    }

    void onIncidentStatusChanged(@ObservesAsync SocIncidentStatusChangedEvent event) {
        String eventType = mapIncidentStatus(event.previousStatus(), event.newStatus());
        if (eventType == null) return;

        var notification = new SocIncidentNotification(
            eventType, event.tenancyId(), event.caseId().toString(),
            null, null, null,
            SocNotificationEvents.ACTOR_SYSTEM, null, event.occurredAt());

        publish(notification);
    }

    void onSlaBreach(@Observes(during = jakarta.enterprise.event.TransactionPhase.AFTER_SUCCESS)
                     SlaBreachEvent event) {
        var task = event.context().task();
        if (task.callerRef() == null || !task.callerRef().startsWith(SOC_CALLER_REF_PREFIX)) return;

        String caseId = task.callerRef().substring(SOC_CALLER_REF_PREFIX.length());

        var notification = new SocIncidentNotification(
            SocNotificationEvents.INCIDENT_SLA_BREACHED,
            event.context().tenancyId(), caseId,
            null, null, null,
            SocNotificationEvents.ACTOR_SYSTEM, null, Instant.now());

        publish(notification);
    }

    void onWorkItemEvent(@Observes(during = jakarta.enterprise.event.TransactionPhase.AFTER_SUCCESS)
                         WorkItemLifecycleEvent event) {
        if (event.callerRef() == null || !event.callerRef().startsWith(SOC_CALLER_REF_PREFIX)) return;

        String eventType = mapWorkItemStatus(event.status());
        if (eventType == null) return;

        String caseId = event.callerRef().substring(SOC_CALLER_REF_PREFIX.length());

        var notification = new SocWorkItemNotification(
            eventType, event.tenancyId(), event.workItemId(),
            caseId, event.title(), event.assigneeId(),
            event.candidateGroups(), SocNotificationEvents.ACTOR_SYSTEM,
            Instant.now());

        publish(notification);
    }

    private void publish(Object event) {
        if (dataSourceRegistryInstance.isUnsatisfied()) {
            LOG.debug("DataSourceRegistry unavailable — notification not published");
            return;
        }
        dataSourceRegistryInstance.get()
            .resolveSource(NOTIFICATION_DATASOURCE_PATH, "platform")
            .ifPresent(ds -> ds.add(event));
    }

    private static String mapIncidentStatus(String previousStatus, String newStatus) {
        if (previousStatus == null) return SocNotificationEvents.INCIDENT_CREATED;
        return switch (newStatus) {
            case "ESCALATED" -> SocNotificationEvents.INCIDENT_ESCALATED;
            case "RESOLVED", "FALSE_POSITIVE" -> SocNotificationEvents.INCIDENT_RESOLVED;
            default -> null;
        };
    }

    private static String mapWorkItemStatus(String status) {
        return switch (status) {
            case "PENDING" -> SocNotificationEvents.WORKITEM_CREATED;
            case "ASSIGNED", "IN_PROGRESS" -> SocNotificationEvents.WORKITEM_ASSIGNED;
            case "ESCALATED" -> SocNotificationEvents.WORKITEM_ESCALATED;
            case "COMPLETED" -> SocNotificationEvents.WORKITEM_COMPLETED;
            default -> null;
        };
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl app -am test -Dtest=SocNotificationBridgeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/notification/SocNotificationBridge.java \
  app/src/test/java/io/casehub/soc/engine/notification/SocNotificationBridgeTest.java
git commit -m "feat(#47): add SocNotificationBridge — CDI observers for notification DataSource

Single bridge class observes SocIncidentStatusChangedEvent (async),
SlaBreachEvent and WorkItemLifecycleEvent (after-success). Filters
work item events by callerRef prefix for SOC incident cases.
Registers 8 event types with EventTypeRegistry at startup.

Refs #47"
```

### Task 3: SocNotificationSeeder — SYSTEM subscription seeding

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/notification/SocNotificationSeeder.java`
- Test: `app/src/test/java/io/casehub/soc/engine/notification/SocNotificationSeederTest.java`

**Interfaces:**
- Consumes: `SocNotificationEvents` (constants from Task 1)
- Consumes: `Instance<SubscriptionStore>` (platform SPI)
- Consumes: `NotificationTemplate`, `NotificationSeverity`, `SubscriptionInput`, `SubscriptionQuery` (platform API)
- Produces: SYSTEM-scope subscriptions in `SubscriptionStore` (consumed by SubscriptionEngine at runtime)

- [ ] **Step 1: Write failing test for subscription seeding**

```java
package io.casehub.soc.engine.notification;

import io.casehub.platform.api.subscription.*;
import io.casehub.soc.domain.SocNotificationEvents;
import jakarta.enterprise.inject.Instance;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.Optional;
import java.util.stream.Stream;

import static org.junit.jupiter.api.Assertions.*;
import static org.mockito.ArgumentMatchers.any;
import static org.mockito.Mockito.*;

class SocNotificationSeederTest {

    private SocNotificationSeeder seeder;
    private SubscriptionStore subscriptionStore;
    private List<SubscriptionInput> storedInputs;

    @BeforeEach
    void setUp() {
        subscriptionStore = mock(SubscriptionStore.class);
        storedInputs = new ArrayList<>();

        when(subscriptionStore.find(any(SubscriptionQuery.class)))
            .thenReturn(new SubscriptionPage(List.of(), null));
        when(subscriptionStore.store(any(SubscriptionInput.class)))
            .thenAnswer(inv -> {
                storedInputs.add(inv.getArgument(0));
                return mock(Subscription.class);
            });

        Instance<SubscriptionStore> ssInstance = mock(Instance.class);
        when(ssInstance.isUnsatisfied()).thenReturn(false);
        when(ssInstance.get()).thenReturn(subscriptionStore);

        seeder = new SocNotificationSeeder(ssInstance, List.of("tenant-1"));
    }

    @Test
    void seedsSubscriptionsForAllEventTypes() {
        seeder.onStartup(null);

        assertTrue(storedInputs.size() >= 8,
            "Expected at least 8 subscriptions, got " + storedInputs.size());
    }

    @Test
    void allSubscriptionsAreSystemScope() {
        seeder.onStartup(null);

        storedInputs.forEach(input ->
            assertEquals(SubscriptionScope.SYSTEM, input.scope()));
    }

    @Test
    void idempotentWhenSubscriptionsExist() {
        var existingSubs = SocNotificationSeeder.defaultSubscriptions("tenant-1");
        var mockPage = new SubscriptionPage(
            existingSubs.stream()
                .map(input -> mock(Subscription.class, inv -> {
                    if (inv.getMethod().getName().equals("eventType"))
                        return input.eventType();
                    if (inv.getMethod().getName().equals("template"))
                        return input.template();
                    return null;
                }))
                .toList(), null);
        when(subscriptionStore.find(any(SubscriptionQuery.class)))
            .thenReturn(mockPage);

        seeder.onStartup(null);

        assertTrue(storedInputs.isEmpty(), "Should not create duplicates");
    }

    @Test
    void gracefulDegradationWhenStoreUnavailable() {
        Instance<SubscriptionStore> ssInstance = mock(Instance.class);
        when(ssInstance.isUnsatisfied()).thenReturn(true);

        seeder = new SocNotificationSeeder(ssInstance, List.of("tenant-1"));
        assertDoesNotThrow(() -> seeder.onStartup(null));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl app -am test -Dtest=SocNotificationSeederTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — SocNotificationSeeder not found

- [ ] **Step 3: Implement SocNotificationSeeder**

```java
package io.casehub.soc.engine.notification;

import io.casehub.platform.api.notification.NotificationSeverity;
import io.casehub.platform.api.subscription.*;
import io.casehub.soc.domain.SocNotificationEvents;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;
import org.jboss.logging.Logger;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.stream.Collectors;

@ApplicationScoped
public class SocNotificationSeeder {

    private static final Logger LOG = Logger.getLogger(SocNotificationSeeder.class);
    private static final String SYSTEM_OWNER = "SYSTEM";

    private final Instance<SubscriptionStore> subscriptionStoreInstance;
    private final List<String> tenants;

    @Inject
    public SocNotificationSeeder(Instance<SubscriptionStore> subscriptionStoreInstance,
                                 @ConfigProperty(name = "casehub.soc.notification.tenants",
                                     defaultValue = "default") List<String> tenants) {
        this.subscriptionStoreInstance = subscriptionStoreInstance;
        this.tenants = tenants;
    }

    void onStartup(@Observes StartupEvent event) {
        if (subscriptionStoreInstance.isUnsatisfied()) {
            LOG.warn("SubscriptionStore unavailable — SYSTEM subscriptions not seeded");
            return;
        }
        var store = subscriptionStoreInstance.get();
        for (String tenantId : tenants) {
            seedTenant(store, tenantId);
        }
    }

    private void seedTenant(SubscriptionStore store, String tenantId) {
        var existingPage = store.find(new SubscriptionQuery(
            SYSTEM_OWNER, tenantId, SubscriptionScope.SYSTEM, true, null, 500));
        Map<String, Subscription> existingByType = existingPage.subscriptions().stream()
            .collect(Collectors.toMap(Subscription::eventType, s -> s, (a, b) -> a));

        var defaults = defaultSubscriptions(tenantId);
        int created = 0;
        int updated = 0;

        for (SubscriptionInput input : defaults) {
            var existing = existingByType.get(input.eventType());
            if (existing == null) {
                store.store(input);
                created++;
            } else if (!templateMatches(existing.template(), input.template())) {
                store.update(existing.id(), SYSTEM_OWNER, tenantId,
                    new SubscriptionUpdate(null, null, null, null,
                        null, input.template(), null));
                updated++;
            }
        }

        LOG.infof("SOC notification seeding for tenant '%s': %d created, %d updated, %d unchanged",
            tenantId, created, updated, defaults.size() - created - updated);
    }

    private static boolean templateMatches(NotificationTemplate a, NotificationTemplate b) {
        return a.titlePattern().equals(b.titlePattern())
            && java.util.Objects.equals(a.bodyPattern(), b.bodyPattern())
            && a.severity() == b.severity()
            && a.category().equals(b.category())
            && java.util.Objects.equals(a.actionUrlPattern(), b.actionUrlPattern());
    }

    static List<SubscriptionInput> defaultSubscriptions(String tenantId) {
        return List.of(
            sub(tenantId, SocNotificationEvents.INCIDENT_CREATED,
                "Incident Created (Critical)",
                template("{severity} incident — {correlationKey}",
                    "New incident. Tactic: {tactic}. Requires immediate triage.",
                    NotificationSeverity.URGENT, "soc-incident",
                    "/soc/incidents/{caseId}",
                    "incident", "caseId", "actorId"),
                List.of(new NotificationTarget(TargetType.GROUP, "soc-manager"),
                    new NotificationTarget(TargetType.GROUP, "soc-on-call"))),

            sub(tenantId, SocNotificationEvents.INCIDENT_ESCALATED,
                "Incident Escalated",
                template("ESCALATED: incident {caseId}",
                    "Incident escalated to higher tier. Immediate attention required.",
                    NotificationSeverity.URGENT, "soc-incident",
                    "/soc/incidents/{caseId}",
                    "incident", "caseId", "actorId"),
                List.of(new NotificationTarget(TargetType.GROUP, "soc-manager"))),

            sub(tenantId, SocNotificationEvents.INCIDENT_RESOLVED,
                "Incident Resolved",
                template("Incident {caseId} resolved",
                    "Incident closed. Correlation: {correlationKey}.",
                    NotificationSeverity.INFO, "soc-incident",
                    "/soc/incidents/{caseId}",
                    "incident", "caseId", "actorId"),
                List.of(new NotificationTarget(TargetType.ENTITY_WATCHERS, "incident"))),

            sub(tenantId, SocNotificationEvents.INCIDENT_SLA_BREACHED,
                "SLA Breached",
                template("SLA BREACH: incident {caseId}",
                    "Response deadline exceeded. Escalation in progress.",
                    NotificationSeverity.URGENT, "soc-sla",
                    "/soc/incidents/{caseId}",
                    "incident", "caseId", "actorId"),
                List.of(new NotificationTarget(TargetType.GROUP, "soc-manager"))),

            sub(tenantId, SocNotificationEvents.WORKITEM_CREATED,
                "SOC Work Item Created",
                template("Action required: {title}",
                    "New SOC work item awaiting review.",
                    NotificationSeverity.URGENT, "soc-containment",
                    "/soc/workitems/{workItemId}",
                    "workitem", "workItemId", "actorId"),
                List.of(new NotificationTarget(TargetType.EVENT_FIELD, "candidateGroups"))),

            sub(tenantId, SocNotificationEvents.WORKITEM_ASSIGNED,
                "SOC Work Item Assigned",
                template("Assigned: {title}",
                    "You have been assigned a SOC investigation task.",
                    NotificationSeverity.WARNING, "soc-workitem",
                    "/soc/workitems/{workItemId}",
                    "workitem", "workItemId", "actorId"),
                List.of(new NotificationTarget(TargetType.EVENT_FIELD, "assigneeId"))),

            sub(tenantId, SocNotificationEvents.WORKITEM_ESCALATED,
                "SOC Work Item Escalated",
                template("ESCALATED: {title}",
                    "SOC work item escalated via SLA breach.",
                    NotificationSeverity.URGENT, "soc-workitem",
                    "/soc/workitems/{workItemId}",
                    "workitem", "workItemId", "actorId"),
                List.of(new NotificationTarget(TargetType.EVENT_FIELD, "candidateGroups"))),

            sub(tenantId, SocNotificationEvents.WORKITEM_COMPLETED,
                "SOC Work Item Completed",
                template("Completed: {title}",
                    "SOC work item has been completed.",
                    NotificationSeverity.INFO, "soc-workitem",
                    "/soc/workitems/{workItemId}",
                    "workitem", "workItemId", "actorId"),
                List.of(new NotificationTarget(TargetType.EVENT_FIELD, "assigneeId")))
        );
    }

    private static SubscriptionInput sub(String tenantId, String eventType, String name,
                                         NotificationTemplate template,
                                         List<NotificationTarget> targets) {
        return new SubscriptionInput(SYSTEM_OWNER, tenantId, name, eventType,
            List.of(), targets, false, template, true, SubscriptionScope.SYSTEM);
    }

    private static NotificationTemplate template(String title, String body,
                                                  NotificationSeverity severity,
                                                  String category, String actionUrl,
                                                  String entityType,
                                                  String entityIdField,
                                                  String actorIdField) {
        return new NotificationTemplate(title, body, severity, category,
            actionUrl, entityType, entityIdField, actorIdField);
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl app -am test -Dtest=SocNotificationSeederTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: BUILD SUCCESS — all existing tests still pass

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/notification/SocNotificationSeeder.java \
  app/src/test/java/io/casehub/soc/engine/notification/SocNotificationSeederTest.java
git commit -m "feat(#47): add SocNotificationSeeder — SYSTEM subscription seeding at startup

Seeds 8 SYSTEM-scope subscriptions per tenant with notification
templates. Idempotent via content comparison — creates missing subs,
updates changed templates, skips matching ones.

Refs #47"
```

---

## Batch 3: Integration verification

### Task 4: End-to-end integration test

**Files:**
- Test: `app/src/test/java/io/casehub/soc/engine/notification/SocNotificationIntegrationTest.java`

**Interfaces:**
- Consumes: `SocNotificationBridge` (Task 2), `SocNotificationSeeder` (Task 3)
- Consumes: `NotificationStore` (platform SPI — verifies notifications were created)
- Consumes: CDI `Event<SocIncidentStatusChangedEvent>` (fires test events)

- [ ] **Step 1: Write integration test**

```java
package io.casehub.soc.engine.notification;

import io.casehub.platform.api.subscription.EventTypeRegistry;
import io.casehub.soc.domain.SocNotificationEvents;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.junit.jupiter.api.Assertions.*;

@QuarkusTest
class SocNotificationIntegrationTest {

    @Inject
    EventTypeRegistry eventTypeRegistry;

    @Test
    void allSocEventTypesRegistered() {
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.INCIDENT_CREATED).orElse(null),
            "INCIDENT_CREATED not registered");
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.INCIDENT_ESCALATED).orElse(null),
            "INCIDENT_ESCALATED not registered");
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.INCIDENT_RESOLVED).orElse(null),
            "INCIDENT_RESOLVED not registered");
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.INCIDENT_SLA_BREACHED).orElse(null),
            "INCIDENT_SLA_BREACHED not registered");
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.WORKITEM_CREATED).orElse(null),
            "WORKITEM_CREATED not registered");
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.WORKITEM_ASSIGNED).orElse(null),
            "WORKITEM_ASSIGNED not registered");
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.WORKITEM_ESCALATED).orElse(null),
            "WORKITEM_ESCALATED not registered");
        assertNotNull(eventTypeRegistry.resolve(SocNotificationEvents.WORKITEM_COMPLETED).orElse(null),
            "WORKITEM_COMPLETED not registered");
    }

    @Test
    void incidentCreatedDescriptorHasExpectedFields() {
        var descriptor = eventTypeRegistry.resolve(SocNotificationEvents.INCIDENT_CREATED)
            .orElseThrow();
        assertEquals("Incident Created", descriptor.displayName());
        assertFalse(descriptor.fields().isEmpty(), "Should have filterable fields");
        assertTrue(descriptor.fields().stream()
            .anyMatch(f -> "severity".equals(f.name())));
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode -pl app -am test -Dtest=SocNotificationIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — event types registered at startup, discoverable via registry

- [ ] **Step 3: Run full build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add app/src/test/java/io/casehub/soc/engine/notification/SocNotificationIntegrationTest.java
git commit -m "test(#47): add notification pipeline integration test — event type registration

Verifies all 8 SOC event types are registered with EventTypeRegistry
at startup with correct display names and filterable fields.

Refs #47"
```

---

## References

- [2026-09-05-soc-notification-pipeline-design.md] — design spec this plan implements
- [SocIncidentStatusChangedEvent.java] — existing CDI event (api/src/main/java/io/casehub/soc/domain/)
- [SocIncidentStatusObserver.java] — fires status events via fireAsync() (app/src/main/java/io/casehub/soc/engine/)
- [SocContainmentCommitmentBridge.java] — existing CDI bridge pattern in SOC (app/src/main/java/io/casehub/soc/engine/mesh/)
- [WorkItemSubscriptionBridge] — platform bridge pattern precedent (casehub-platform subscriptions module)
- [SubscriptionEngine.java] — platform subscription matching engine
- [NotificationDispatcher.java] — platform notification dispatch pipeline
- [TemplateResolver.java] — platform {field} placeholder resolver
- [GitHub #47] — SOC notification pipeline epic
