# Helpdesk UI Upgrade Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #412 — Helpdesk UI — blocks-ui components, runtime state visualization, SSE push
**Issue group:** #408, #412

**Goal:** Replace the vanilla Lit helpdesk dashboard with platform components (blocks-ui, pages) and real-time push via the pages-push WebSocket wire protocol, with a step-by-step scenario controller.

**Architecture:** Backend fires CDI events on ticket state changes, observed synchronously by `TicketPushObserver` which bridges to `EventBroadcaster.broadcast()`. A `@WebSocket` endpoint provides the transport. Frontend uses per-topic `EventStreamController` instances from `@casehubio/pages-component` with `blocks-kpi-metric-row`, `pages-table`, and `blocks-timeline` for visualization.

**Tech Stack:** Java 21 / Quarkus 3.32 / pages-push / Lit / Vite / blocks-ui / pages-component / pages-table

## Global Constraints

- Java 21, Quarkus 3.32.2, CaseHub platform 0.2-SNAPSHOT
- Synchronous CDI events (`@Observes` + `Event.fire()`) — ordering matters for pipeline visualization
- Per-topic `EventStreamController` instances, not wildcard subscription
- `EventStreamPool` reuses the same WebSocket connection across controllers
- Package names must be verified against each package's `package.json` `name` field (GE-20260803-17fc03)
- Examples repo sources are on branch `issue-408-scenario-engine` — checkout that branch before starting

---

### Task 1: CDI Event Types and TicketService Integration

**Files:**
- Create: `src/main/java/io/casehub/examples/helpdesk/event/TicketEvent.java`
- Create: `src/main/java/io/casehub/examples/helpdesk/event/NotificationEvent.java`
- Modify: `src/main/java/io/casehub/examples/helpdesk/TicketService.java`
- Modify: `src/main/java/io/casehub/examples/helpdesk/NotificationService.java`
- Test: `src/test/java/io/casehub/examples/helpdesk/event/TicketEventFiringTest.java`

**Interfaces:**
- Consumes: `Ticket` record (existing), `TicketStatus` enum (existing)
- Produces: `TicketEvent(Type type, Ticket ticket)` with `Type { CREATED, CLASSIFIED, ASSIGNED, RESOLVED }`, `NotificationEvent(String to, String message)`. Both fired synchronously via `Event.fire()`.

- [ ] **Step 1: Write the TicketEvent record**

```java
package io.casehub.examples.helpdesk.event;

import io.casehub.examples.helpdesk.model.Ticket;

public record TicketEvent(Type type, Ticket ticket) {
    public enum Type { CREATED, CLASSIFIED, ASSIGNED, RESOLVED }
}
```

- [ ] **Step 2: Write the NotificationEvent record**

```java
package io.casehub.examples.helpdesk.event;

public record NotificationEvent(String to, String message) {}
```

- [ ] **Step 3: Write the failing test for TicketService event firing**

```java
package io.casehub.examples.helpdesk.event;

import io.casehub.examples.helpdesk.TicketService;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import jakarta.inject.Singleton;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.concurrent.CopyOnWriteArrayList;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class TicketEventFiringTest {

    @Inject TicketService ticketService;
    @Inject EventCaptor captor;

    @BeforeEach
    void reset() {
        captor.events.clear();
    }

    @Test
    void create_fires_created_event() {
        var ticket = ticketService.create("Laptop broken", "Screen cracked", "alice");
        assertThat(captor.events).hasSize(1);
        assertThat(captor.events.get(0).type()).isEqualTo(TicketEvent.Type.CREATED);
        assertThat(captor.events.get(0).ticket().id()).isEqualTo(ticket.id());
    }

    @Test
    void classify_fires_classified_event() {
        var ticket = ticketService.create("Laptop broken", "Screen cracked", "alice");
        captor.events.clear();
        ticketService.classify(ticket.id(),
                io.casehub.examples.helpdesk.model.TicketCategory.HARDWARE,
                io.casehub.examples.helpdesk.model.TicketPriority.HIGH);
        assertThat(captor.events).hasSize(1);
        assertThat(captor.events.get(0).type()).isEqualTo(TicketEvent.Type.CLASSIFIED);
    }

    @Test
    void assign_fires_assigned_event() {
        var ticket = ticketService.create("Laptop broken", "Screen cracked", "alice");
        captor.events.clear();
        ticketService.assign(ticket.id(), "hw-specialist");
        assertThat(captor.events).hasSize(1);
        assertThat(captor.events.get(0).type()).isEqualTo(TicketEvent.Type.ASSIGNED);
    }

    @Test
    void resolve_fires_resolved_event() {
        var ticket = ticketService.create("Laptop broken", "Screen cracked", "alice");
        captor.events.clear();
        ticketService.resolve(ticket.id(), "Replaced screen");
        assertThat(captor.events).hasSize(1);
        assertThat(captor.events.get(0).type()).isEqualTo(TicketEvent.Type.RESOLVED);
    }

    @Singleton
    static class EventCaptor {
        final List<TicketEvent> events = new CopyOnWriteArrayList<>();

        void onTicketEvent(@Observes TicketEvent event) {
            events.add(event);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn test -pl helpdesk -Dtest=TicketEventFiringTest -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: FAIL — TicketService doesn't fire events yet.

- [ ] **Step 5: Modify TicketService to inject and fire events**

Add `@Inject Event<TicketEvent> ticketEvent` field. After each state-changing operation, fire the corresponding event synchronously. Each method must re-read the ticket from the map after mutation (since `computeIfPresent` returns the new value):

```java
@Inject
Event<TicketEvent> ticketEvent;

public Ticket create(String subject, String description, String customerRef) {
    var ticket = new Ticket(/* ... existing ... */);
    tickets.put(ticket.id(), ticket);
    ticketEvent.fire(new TicketEvent(TicketEvent.Type.CREATED, ticket));
    return ticket;
}

public void classify(UUID ticketId, TicketCategory category, TicketPriority priority) {
    var updated = tickets.computeIfPresent(ticketId, (id, t) -> t.withClassification(category, priority));
    if (updated != null) {
        ticketEvent.fire(new TicketEvent(TicketEvent.Type.CLASSIFIED, updated));
    }
}

public void assign(UUID ticketId, String assigneeId) {
    var updated = tickets.computeIfPresent(ticketId, (id, t) -> t.withAssignee(assigneeId));
    if (updated != null) {
        ticketEvent.fire(new TicketEvent(TicketEvent.Type.ASSIGNED, updated));
    }
}

public Ticket resolve(UUID ticketId, String resolution) {
    var updated = tickets.computeIfPresent(ticketId, (id, t) -> t.withResolution(resolution));
    if (updated != null) {
        ticketEvent.fire(new TicketEvent(TicketEvent.Type.RESOLVED, updated));
    }
    return updated;
}
```

- [ ] **Step 6: Modify NotificationService to fire NotificationEvent**

Add `@Inject Event<NotificationEvent> notificationEvent`. Fire after sending:

```java
@Inject
Event<NotificationEvent> notificationEvent;

public void notify(String customerRef, String message) {
    chatPlatform.messaging().send(new ChatChannelRef(customerRef), new ChatContent(message));
    sent.add(new SentNotification(customerRef, message, Instant.now()));
    notificationEvent.fire(new NotificationEvent(customerRef, message));
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `mvn test -pl helpdesk -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: ALL PASS (including existing tests — CDI events are additive)

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/src
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): CDI event types and synchronous event firing from TicketService/NotificationService

Refs casehubio/parent#412"
```

---

### Task 2: Push Infrastructure — WebSocket Endpoint and SessionSender

**Files:**
- Modify: `pom.xml` — add quarkus-websockets-next, casehub-pages-push, casehub-pages-push-runtime
- Create: `src/main/java/io/casehub/examples/helpdesk/push/HelpdeskPushEndpoint.java`
- Create: `src/main/java/io/casehub/examples/helpdesk/push/HelpdeskSessionSender.java`
- Create: `src/main/java/io/casehub/examples/helpdesk/push/ConnectionRegistry.java`
- Test: `src/test/java/io/casehub/examples/helpdesk/push/PushEndpointTest.java`

**Interfaces:**
- Consumes: `PushRequest.parse(String)` (from pages-push), `TopicRegistry.listen(connId, topics)` / `unlisten(connId, topics)` / `removeConnection(connId)` (from pages-push-runtime CDI), `EventStore.replay(topic, sinceSeq, limit)` (from pages-push-runtime CDI), `PushMessage.event(topic, payloadJson, seq)` / `PushMessage.ack(id, topics, gaps)` / `PushMessage.error(id, message)` (from pages-push)
- Produces: `ConnectionRegistry` (shared connection map), `HelpdeskSessionSender` implements `SessionSender`, WebSocket endpoint at `/push`

- [ ] **Step 1: Add dependencies to pom.xml**

Add to the `<dependencies>` section of `helpdesk/pom.xml`:

```xml
<dependency>
    <groupId>io.quarkus</groupId>
    <artifactId>quarkus-websockets-next</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-push</artifactId>
    <version>${casehub.version}</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-pages-push-runtime</artifactId>
    <version>${casehub.version}</version>
</dependency>
```

- [ ] **Step 2: Write ConnectionRegistry (shared connection map)**

```java
package io.casehub.examples.helpdesk.push;

import io.quarkus.websocket.next.WebSocketConnection;
import jakarta.enterprise.context.ApplicationScoped;

import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ConnectionRegistry {

    private final Map<String, WebSocketConnection> connections = new ConcurrentHashMap<>();

    public void register(String connectionId, WebSocketConnection connection) {
        connections.put(connectionId, connection);
    }

    public void unregister(String connectionId) {
        connections.remove(connectionId);
    }

    public WebSocketConnection get(String connectionId) {
        return connections.get(connectionId);
    }
}
```

- [ ] **Step 3: Write HelpdeskSessionSender**

```java
package io.casehub.examples.helpdesk.push;

import io.casehub.pages.push.SessionSender;
import io.quarkus.websocket.next.WebSocketConnection;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

@ApplicationScoped
public class HelpdeskSessionSender implements SessionSender {

    @Inject
    ConnectionRegistry registry;

    @Override
    public void send(String connectionId, String message) {
        WebSocketConnection conn = registry.get(connectionId);
        if (conn != null && !conn.isClosed()) {
            conn.sendTextAndAwait(message);
        }
    }
}
```

- [ ] **Step 4: Write the failing integration test**

```java
package io.casehub.examples.helpdesk.push;

import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.WebSocket;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class PushEndpointTest {

    @TestHTTPResource("/push")
    URI pushUri;

    @Test
    void client_connects_and_receives_ack_on_listen() throws Exception {
        var messages = new CopyOnWriteArrayList<String>();
        var latch = new CountDownLatch(1);

        var ws = HttpClient.newHttpClient().newWebSocketBuilder()
                .buildAsync(pushUri, new WebSocket.Listener() {
                    @Override
                    public CompletionStage<?> onText(WebSocket webSocket, CharSequence data, boolean last) {
                        messages.add(data.toString());
                        latch.countDown();
                        webSocket.request(1);
                        return null;
                    }
                }).join();

        ws.sendText("{\"op\":\"listen\",\"id\":\"req-1\",\"topics\":[\"helpdesk:tickets\"]}", true);

        assertThat(latch.await(5, TimeUnit.SECONDS)).isTrue();
        assertThat(messages).hasSize(1);
        assertThat(messages.get(0)).contains("\"op\":\"ack\"");
        assertThat(messages.get(0)).contains("\"id\":\"req-1\"");

        ws.sendClose(WebSocket.NORMAL_CLOSURE, "done").join();
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn test -pl helpdesk -Dtest=PushEndpointTest -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: FAIL — no WebSocket endpoint exists yet.

- [ ] **Step 6: Write HelpdeskPushEndpoint**

```java
package io.casehub.examples.helpdesk.push;

import io.casehub.pages.push.EventStore;
import io.casehub.pages.push.PushMessage;
import io.casehub.pages.push.PushRequest;
import io.casehub.pages.push.StoredEvent;
import io.casehub.pages.push.TopicRegistry;
import io.quarkus.websocket.next.OnClose;
import io.quarkus.websocket.next.OnOpen;
import io.quarkus.websocket.next.OnTextMessage;
import io.quarkus.websocket.next.WebSocket;
import io.quarkus.websocket.next.WebSocketConnection;
import jakarta.inject.Inject;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

@WebSocket(path = "/push")
public class HelpdeskPushEndpoint {

    @Inject ConnectionRegistry connectionRegistry;
    @Inject TopicRegistry topicRegistry;
    @Inject EventStore eventStore;

    @OnOpen
    void onOpen(WebSocketConnection connection) {
        String connId = UUID.randomUUID().toString();
        connection.userData().put("connId", connId);
        connectionRegistry.register(connId, connection);
    }

    @OnTextMessage
    void onMessage(WebSocketConnection connection, String message) {
        String connId = (String) connection.userData().get("connId");
        PushRequest request = PushRequest.parse(message);

        switch (request) {
            case PushRequest.Listen listen -> {
                topicRegistry.listen(connId, listen.topics());

                List<String> gaps = new ArrayList<>();
                for (var entry : listen.since().entrySet()) {
                    List<StoredEvent> events = eventStore.replay(entry.getKey(), entry.getValue(), 1000);
                    if (events.isEmpty() && entry.getValue() > 0) {
                        gaps.add(entry.getKey());
                    }
                    for (var stored : events) {
                        connection.sendTextAndAwait(
                                PushMessage.event(stored.topic(), stored.payloadJson(), stored.seq()));
                    }
                }

                connection.sendTextAndAwait(
                        PushMessage.ack(listen.id(), listen.topics(), gaps));
            }
            case PushRequest.Unlisten unlisten -> {
                topicRegistry.unlisten(connId, unlisten.topics());
                connection.sendTextAndAwait(
                        PushMessage.ack(unlisten.id(), unlisten.topics(), List.of()));
            }
            default -> connection.sendTextAndAwait(
                    PushMessage.error(request.id(), "Unsupported op: " + request.op()));
        }
    }

    @OnClose
    void onClose(WebSocketConnection connection) {
        String connId = (String) connection.userData().get("connId");
        if (connId != null) {
            topicRegistry.removeConnection(connId);
            connectionRegistry.unregister(connId);
        }
    }
}
```

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -pl helpdesk -Dtest=PushEndpointTest -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: PASS

- [ ] **Step 8: Run all tests**

Run: `mvn test -pl helpdesk -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: ALL PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): pages-push WebSocket endpoint with SessionSender and ConnectionRegistry

Refs casehubio/parent#412"
```

---

### Task 3: TicketPushObserver — CDI Event to Push Bridge

**Files:**
- Create: `src/main/java/io/casehub/examples/helpdesk/push/TicketPushObserver.java`
- Test: `src/test/java/io/casehub/examples/helpdesk/push/TicketPushObserverTest.java`

**Interfaces:**
- Consumes: `TicketEvent` (Task 1), `NotificationEvent` (Task 1), `EventBroadcaster` (pages-push-runtime CDI), `TicketService.findAll()` (for metrics aggregation)
- Produces: Push events on topics `helpdesk:tickets`, `helpdesk:notifications`, `helpdesk:metrics`

- [ ] **Step 1: Write the failing test**

```java
package io.casehub.examples.helpdesk.push;

import io.casehub.examples.helpdesk.TicketService;
import io.casehub.examples.helpdesk.event.TicketEvent;
import io.casehub.examples.helpdesk.model.TicketCategory;
import io.casehub.examples.helpdesk.model.TicketPriority;
import io.quarkus.test.common.http.TestHTTPResource;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.net.URI;
import java.net.http.HttpClient;
import java.net.http.WebSocket;
import java.util.concurrent.CompletionStage;
import java.util.concurrent.CopyOnWriteArrayList;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class TicketPushObserverTest {

    @TestHTTPResource("/push")
    URI pushUri;

    @Inject
    TicketService ticketService;

    @Test
    void ticket_creation_broadcasts_to_connected_client() throws Exception {
        var messages = new CopyOnWriteArrayList<String>();
        var ackLatch = new CountDownLatch(1);
        var eventLatch = new CountDownLatch(2); // ticket event + metrics event

        var ws = HttpClient.newHttpClient().newWebSocketBuilder()
                .buildAsync(pushUri, new WebSocket.Listener() {
                    @Override
                    public CompletionStage<?> onText(WebSocket webSocket, CharSequence data, boolean last) {
                        String msg = data.toString();
                        messages.add(msg);
                        if (msg.contains("\"op\":\"ack\"")) ackLatch.countDown();
                        if (msg.contains("\"op\":\"event\"")) eventLatch.countDown();
                        webSocket.request(1);
                        return null;
                    }
                }).join();

        ws.sendText("{\"op\":\"listen\",\"id\":\"r1\",\"topics\":[\"helpdesk:tickets\",\"helpdesk:metrics\"]}", true);
        assertThat(ackLatch.await(5, TimeUnit.SECONDS)).isTrue();

        ticketService.create("Test ticket", "Test description", "bob");

        assertThat(eventLatch.await(5, TimeUnit.SECONDS)).isTrue();

        var ticketEvents = messages.stream()
                .filter(m -> m.contains("helpdesk:tickets")).toList();
        assertThat(ticketEvents).isNotEmpty();
        assertThat(ticketEvents.get(0)).contains("CREATED");

        var metricsEvents = messages.stream()
                .filter(m -> m.contains("helpdesk:metrics")).toList();
        assertThat(metricsEvents).isNotEmpty();
        assertThat(metricsEvents.get(0)).contains("\"total\":1");

        ws.sendClose(WebSocket.NORMAL_CLOSURE, "done").join();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn test -pl helpdesk -Dtest=TicketPushObserverTest -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: FAIL — no observer bridges events to push yet.

- [ ] **Step 3: Write TicketPushObserver**

```java
package io.casehub.examples.helpdesk.push;

import io.casehub.examples.helpdesk.NotificationService;
import io.casehub.examples.helpdesk.TicketService;
import io.casehub.examples.helpdesk.event.NotificationEvent;
import io.casehub.examples.helpdesk.event.TicketEvent;
import io.casehub.examples.helpdesk.model.TicketStatus;
import io.casehub.pages.push.EventBroadcaster;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;

import java.util.Map;
import java.util.Set;

@ApplicationScoped
public class TicketPushObserver {

    private static final Set<TicketStatus> OPEN_STATUSES =
            Set.of(TicketStatus.OPEN, TicketStatus.TRIAGED, TicketStatus.ASSIGNED);

    @Inject EventBroadcaster broadcaster;
    @Inject TicketService ticketService;
    @Inject NotificationService notificationService;

    void onTicketEvent(@Observes TicketEvent event) {
        broadcaster.broadcast("helpdesk:tickets", event);
        broadcastMetrics();
    }

    void onNotification(@Observes NotificationEvent event) {
        broadcaster.broadcast("helpdesk:notifications", event);
        broadcastMetrics();
    }

    private void broadcastMetrics() {
        var all = ticketService.findAll();
        long open = all.stream().filter(t -> OPEN_STATUSES.contains(t.status())).count();
        long resolved = all.stream().filter(t -> t.status() == TicketStatus.RESOLVED
                || t.status() == TicketStatus.CLOSED).count();
        long notified = notificationService.getSentNotifications().size();
        broadcaster.broadcast("helpdesk:metrics", Map.of(
                "total", all.size(),
                "open", open,
                "resolved", resolved,
                "notified", notified
        ));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn test -pl helpdesk -Dtest=TicketPushObserverTest -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: PASS

- [ ] **Step 5: Run all tests**

Run: `mvn test -pl helpdesk -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/src
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): TicketPushObserver bridges CDI events to pages-push EventBroadcaster

Refs casehubio/parent#412"
```

---

### Task 4: Frontend — Push Connection and KPI Metrics

**Files:**
- Modify: `src/main/webui/package.json` — add @casehubio dependencies
- Modify: `src/main/webui/vite.config.ts` — add Vite aliases for @casehubio packages
- Modify: `src/main/webui/src/helpdesk-app.ts` — replace polling with push, add kpi-metric-row
- Test: E2E verification via `yarn dev` + browser

**Interfaces:**
- Consumes: WebSocket endpoint at `/push` (Task 2), `EventStreamController` from `@casehubio/pages-component`, `blocks-kpi-metric-row` from blocks-ui
- Produces: Push-driven metrics display replacing the polling metrics

- [ ] **Step 1: Add dependencies to package.json**

Verify package names first by checking each package's `package.json` `name` field. Add to `devDependencies` with `portal:` resolutions pointing to source trees.

- [ ] **Step 2: Configure Vite aliases**

Add aliases in `vite.config.ts` following the blocks-ui-examples pattern — resolve `@casehubio/*` from source trees:

```typescript
import { defineConfig } from 'vite';

export default defineConfig({
  resolve: {
    alias: {
      // Add @casehubio/* aliases pointing to source packages
    },
  },
  server: {
    proxy: {
      '/tickets': 'http://localhost:8090',
      '/scenario': 'http://localhost:8090',
      '/push': { target: 'ws://localhost:8090', ws: true },
    },
  },
});
```

- [ ] **Step 3: Replace polling with EventStreamController**

Remove `_pollTimer`, `_poll()`, `connectedCallback` polling, `disconnectedCallback` timer cleanup. Add three `EventStreamController` instances:

```typescript
import { EventStreamController } from '@casehubio/pages-component';

// In class:
private _ticketPush = new EventStreamController<TicketEvent>(this, '/push', 'helpdesk:tickets');
private _notifPush = new EventStreamController<NotificationEvent>(this, '/push', 'helpdesk:notifications');
private _metricsPush = new EventStreamController<MetricsSnapshot>(this, '/push', 'helpdesk:metrics');
```

- [ ] **Step 4: Replace hand-coded metrics with blocks-kpi-metric-row**

Replace the `.metrics` div with:

```typescript
get _metrics(): MetricsSnapshot {
  return this._metricsPush.latest ?? { total: 0, open: 0, resolved: 0, notified: 0 };
}

get _metricDefs(): MetricDefinition[] {
  return [
    { key: 'total', value: this._metrics.total, label: 'Total' },
    { key: 'open', value: this._metrics.open, label: 'Open', status: 'warning' },
    { key: 'resolved', value: this._metrics.resolved, label: 'Resolved', status: 'normal' },
    { key: 'notified', value: this._metrics.notified, label: 'Notified' },
  ];
}
```

In template:
```html
<blocks-kpi-metric-row .metrics=${this._metricDefs} columns="4" density="compact">
</blocks-kpi-metric-row>
```

- [ ] **Step 5: Verify in browser**

Start backend: `mvn quarkus:dev -pl helpdesk -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Start frontend: `yarn dev` (in webui directory)
Open browser, bootstrap data, submit ticket — verify metrics update via push (no polling).

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/src/main/webui
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): push-driven KPI metrics with blocks-kpi-metric-row, remove polling

Refs casehubio/parent#412"
```

---

### Task 5: Frontend — Ticket Table with pages-table

**Files:**
- Modify: `src/main/webui/src/helpdesk-app.ts` — replace HTML table with pages-table

**Interfaces:**
- Consumes: `_ticketPush.all` (from Task 4), `pages-table` component
- Produces: Push-driven interactive data table with sorting, filtering, badges

- [ ] **Step 1: Replace HTML table with pages-table**

Derive ticket state from push event history:

```typescript
get _tickets(): Ticket[] {
  const map = new Map<string, Ticket>();
  for (const event of this._ticketPush.all) {
    map.set(event.ticket.id, event.ticket);
  }
  return [...map.values()];
}
```

Replace the `<table>` block with `<pages-data-table>` (tag name from `@casehubio/pages-table` — the component was renamed from `pages-data-table`; verify the tag name from the package source):

```html
<pages-data-table
  .data=${this._tickets}
  .columnConfig=${this._ticketColumns}
></pages-data-table>
```

Define column config with badge renderers for Status and Priority columns.

- [ ] **Step 2: Verify in browser**

Start backend + frontend. Submit tickets, verify the table updates via push with sorting and filtering.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/src/main/webui
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): replace HTML table with push-driven pages-data-table

Refs casehubio/parent#412"
```

---

### Task 6: Frontend — Pipeline Timeline with blocks-timeline

**Files:**
- Create: `src/main/webui/src/pipeline-strategy.ts`
- Modify: `src/main/webui/src/helpdesk-app.ts` — add blocks-timeline

**Interfaces:**
- Consumes: `_tickets` (from Task 5), `TimelineStrategy` interface from blocks-timeline
- Produces: `HelpdeskPipelineStrategy`, per-ticket pipeline visualization

- [ ] **Step 1: Write HelpdeskPipelineStrategy**

```typescript
import type { TimelineStrategy, TimelineNode, Layout } from '@casehubio/blocks-ui-blocks-timeline';

const STAGES = ['created', 'classified', 'assigned', 'resolved'] as const;

const STATUS_TO_STAGE: Record<string, number> = {
  OPEN: 0,
  TRIAGED: 1,
  ASSIGNED: 2,
  RESOLVED: 3,
  CLOSED: 3,
};

export class HelpdeskPipelineStrategy implements TimelineStrategy {
  defaultLayout: Layout = 'vertical';

  toNodes(tickets: unknown): TimelineNode[] {
    const ticketList = tickets as Array<{
      id: string; subject: string; status: string;
      createdAt: string; resolvedAt: string | null; assigneeId: string | null;
    }>;
    return ticketList.flatMap(ticket => {
      const completedIdx = STATUS_TO_STAGE[ticket.status] ?? 0;
      return STAGES.map((stage, i) => ({
        key: `${ticket.id}:${stage}`,
        label: `${ticket.subject} — ${stage}`,
        status: (i < completedIdx ? 'completed'
              : i === completedIdx ? 'active'
              : 'pending') as TimelineNode['status'],
        timestamp: stage === 'created' ? ticket.createdAt
                 : stage === 'resolved' ? (ticket.resolvedAt ?? undefined)
                 : undefined,
        actor: stage === 'assigned' ? (ticket.assigneeId ?? undefined) : undefined,
        category: ticket.id,
      }));
    });
  }
}
```

- [ ] **Step 2: Add blocks-timeline to the template**

```typescript
import { HelpdeskPipelineStrategy } from './pipeline-strategy.js';

// In class:
private _pipelineStrategy = new HelpdeskPipelineStrategy();
```

In template:
```html
<blocks-timeline .data=${this._tickets} .strategy=${this._pipelineStrategy}></blocks-timeline>
```

- [ ] **Step 3: Verify in browser**

Submit multiple tickets, verify the timeline shows per-ticket pipeline progression. Resolve a ticket, verify its nodes update to 'completed'.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/src/main/webui
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): pipeline timeline with per-ticket stage visualization via blocks-timeline

Refs casehubio/parent#412"
```

---

### Task 7: Scenario Controller Panel

**Files:**
- Create: `src/main/webui/src/scenarios/help-desk-basic.ts` — embedded scenario steps
- Modify: `src/main/webui/src/helpdesk-app.ts` — add scenario controller panel with split layout

**Interfaces:**
- Consumes: `_ticketPush` (from Task 4), REST endpoints `/scenario/bootstrap/helpdesk` and `/scenario/inject/chat`, pages split container
- Produces: Step-by-step scenario controller with push observation, standalone mode

- [ ] **Step 1: Define scenario steps module**

```typescript
export interface ScenarioStep {
  label: string;
  description: string;
  action: 'bootstrap' | 'submit' | 'resolve' | 'wait';
  params?: Record<string, unknown>;
  previewText?: string;
}

export const HELPDESK_SCENARIO: ScenarioStep[] = [
  {
    label: 'Load classification data',
    description: 'Bootstrap the ticket classifier with keyword → category mappings.',
    action: 'bootstrap',
    params: {
      ticketClassifications: [
        { match: 'laptop', category: 'HARDWARE', priority: 'HIGH' },
        { match: 'password', category: 'ACCESS', priority: 'LOW' },
        { match: 'vpn', category: 'ACCESS', priority: 'HIGH' },
      ],
    },
  },
  {
    label: 'Submit: laptop issue',
    description: 'A user reports a broken laptop. The system creates a ticket, classifies it as HARDWARE/HIGH, and assigns to hw-specialist.',
    action: 'submit',
    params: { from: 'alice', channelId: 'support', text: 'My laptop screen is broken and I cannot work' },
    previewText: 'My laptop screen is broken and I cannot work',
  },
  {
    label: 'Submit: password reset',
    description: 'A user needs a password reset. Classified as ACCESS/LOW, assigned to access-specialist.',
    action: 'submit',
    params: { from: 'bob', channelId: 'support', text: 'I forgot my password and cannot log in' },
    previewText: 'I forgot my password and cannot log in',
  },
  {
    label: 'Resolve: laptop issue',
    description: 'The hardware specialist resolves the laptop ticket. The customer receives a notification.',
    action: 'resolve',
    previewText: 'Replaced screen — new laptop model deployed',
  },
];
```

- [ ] **Step 2: Add scenario controller to helpdesk-app**

Add scenario state management, step rendering, action execution, and push observation for automated stage tracking. Include click feedback CSS (`:active` transforms). Add `?standalone` query parameter support.

The controller uses temporal correlation: after submit, record `_ticketPush.all.length`, watch for the next CREATED event beyond that index to get the ticket ID, then watch for CLASSIFIED and ASSIGNED events with that ID.

- [ ] **Step 3: Add split layout**

Use pages split container to create a resizable split between the dashboard (left) and scenario controller (right). The controller panel renders with step indicator, text preview area, and Submit/Next buttons.

- [ ] **Step 4: Verify in browser**

Full scenario walkthrough: bootstrap → submit laptop → observe auto-classify → observe auto-assign → submit password → resolve laptop → verify notification. Check step indicator updates, text preview, click feedback, and standalone mode (`?standalone`).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/src/main/webui
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): scenario controller panel with step-by-step pacing and push observation

Refs casehubio/parent#412"
```

---

### Task 8: E2E Verification and Polish

**Files:**
- Modify: `src/main/webui/src/helpdesk-app.ts` — CSS polish, click feedback, layout refinements
- No new test files — manual E2E verification through browser

**Interfaces:**
- Consumes: All previous tasks
- Produces: Polished, demonstrable helpdesk UI

- [ ] **Step 1: Full E2E walkthrough**

Start backend + frontend. Run the complete scenario. Verify:
- Metrics update on each event
- Table shows tickets with correct status badges
- Timeline shows per-ticket pipeline progression
- Scenario controller advances through steps
- Standalone mode works in a separate browser window
- Click feedback on all buttons

- [ ] **Step 2: CSS and UX polish**

- Click feedback: `:active` transform and color transitions on all buttons
- Ensure dark theme consistency across blocks-ui components
- Verify responsive behaviour (panel collapse on narrow viewports)
- Check notification display

- [ ] **Step 3: Run all backend tests**

Run: `mvn test -pl helpdesk -f /Users/mdproctor/claude/casehub/examples/pom.xml`
Expected: ALL PASS

- [ ] **Step 4: Final commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add helpdesk/
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#412): UI polish — click feedback, dark theme consistency, layout refinements

Refs casehubio/parent#412"
```
