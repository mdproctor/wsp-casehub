# Cross-Service WorkItem Federation Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #95 — Cross-service WorkItem federation: create in service A, resolve in service B
**Issue group:** #95, #92 (parent epic)

**Goal:** Enable WorkItems created in one casehub-work instance to appear in another instance's inbox, be claimed and resolved there, with lifecycle events flowing bidirectionally.

**Architecture:** Single-writer CQRS — each WorkItem has one owning service. Consuming services maintain shadow WorkItems (read-only projections) synchronized via CloudEvents. Write operations on shadows are proxied to the owner's REST API. A CDI `@Decorator` on `WorkItemStore` prevents accidental local mutation of shadows.

**Tech Stack:** Java 21, Quarkus 3.32, CDI `@Decorator`, CloudEvents, JAX-RS, JPA/Hibernate, Flyway

## Global Constraints

- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build per-module only: `mvn test -pl <module>` (never full build)
- Use `scripts/` helper scripts for timeouts
- Read `docs/GOTCHAS.md` before writing code
- All commits reference #95: `Refs #95 Refs #92`
- Federation module package: `io.casehub.work.federation`
- Client module package: `io.casehub.work.client`
- Flyway: runtime columns → V40; federation tables → V8000+
- IntelliJ MCP mandatory for all .java edits

---

## Batch 1: Foundation — WorkItem Record + Guard

After this batch: the `WorkItem` record has federation fields, all three store backends support them, shadow WorkItems are protected from accidental mutation, and `WorkItemService` is an interface ready for decorator wrapping.

### Task 1: Add federation fields to WorkItem record and store backends

**Files:**
- Modify: `api/src/main/java/io/casehub/work/api/WorkItem.java` — add 3 record fields + Builder setters
- Modify: `api/src/main/java/io/casehub/work/api/spi/WorkItemStore.java` — add `findByOrigin()` default method
- Modify: `runtime/src/main/java/io/casehub/work/runtime/entity/WorkItemEntity.java` — add 3 JPA columns
- Modify: `runtime/src/main/java/io/casehub/work/runtime/mapper/WorkItemEntityMapper.java` — map new fields (exclude `version`, include `originVersion`)
- Modify: `runtime/src/main/java/io/casehub/work/runtime/repository/jpa/JpaWorkItemStore.java` — override `findByOrigin()` with indexed query
- Modify: `persistence-memory/src/main/java/io/casehub/work/memory/InMemoryWorkItemStore.java` — override `findByOrigin()`
- Modify: `persistence-mongodb/src/main/java/io/casehub/work/mongodb/MongoWorkItemStore.java` — override `findByOrigin()`
- Create: `runtime/src/main/resources/db/work/migration/V40__federation_fields.sql`
- Test: `api/src/test/java/io/casehub/work/api/WorkItemFederationFieldsTest.java`
- Test: `runtime/src/test/java/io/casehub/work/runtime/mapper/WorkItemEntityMapperFederationTest.java`

**Interfaces:**
- Consumes: existing `WorkItem` record, `WorkItemStore` SPI, `WorkItemEntityMapper`
- Produces: `WorkItem.originServiceId()`, `WorkItem.originWorkItemId()`, `WorkItem.originVersion()`, `WorkItemStore.findByOrigin(String, UUID)`

- [ ] **Step 1: Write failing test for WorkItem federation fields**

```java
// api/src/test/java/io/casehub/work/api/WorkItemFederationFieldsTest.java
@Test
void shadowWorkItemHasFederationFields() {
    var shadow = WorkItem.builder()
        .id(UUID.randomUUID()).title("Remote task").createdBy("system")
        .status(WorkItemStatus.PENDING).priority(WorkItemPriority.MEDIUM)
        .originServiceId("service-a")
        .originWorkItemId(UUID.randomUUID())
        .originVersion(5L)
        .build();
    assertEquals("service-a", shadow.originServiceId());
    assertNotNull(shadow.originWorkItemId());
    assertEquals(5L, shadow.originVersion());
}

@Test
void localWorkItemHasNullFederationFields() {
    var local = WorkItem.builder()
        .id(UUID.randomUUID()).title("Local task").createdBy("system")
        .status(WorkItemStatus.PENDING).priority(WorkItemPriority.MEDIUM)
        .build();
    assertNull(local.originServiceId());
    assertNull(local.originWorkItemId());
    assertNull(local.originVersion());
}
```

- [ ] **Step 2: Run test — verify FAIL** (fields don't exist yet)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemFederationFieldsTest -pl api
```

- [ ] **Step 3: Add fields to WorkItem record and Builder**

Use `ide_edit_member` to add three record components after `routingExperiences`:

```java
// In WorkItem record: add after routingExperiences (line 56)
String originServiceId,
UUID originWorkItemId,
Long originVersion,
```

Add corresponding Builder fields, setters, `toBuilder()` copy lines, and `build()` constructor args. Follow the exact pattern of existing fields like `templateVersion`.

- [ ] **Step 4: Run test — verify PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=WorkItemFederationFieldsTest -pl api
```

- [ ] **Step 5: Add findByOrigin() default method to WorkItemStore**

```java
// In WorkItemStore interface, after findByCallerRef():
default Optional<WorkItem> findByOrigin(String originServiceId, UUID originWorkItemId) {
    return scanAll().stream()
        .filter(w -> originServiceId.equals(w.originServiceId())
                  && originWorkItemId.equals(w.originWorkItemId()))
        .findFirst();
}
```

- [ ] **Step 6: Add JPA columns to WorkItemEntity**

```java
// Three new fields in WorkItemEntity:
@Column(name = "origin_service_id")
public String originServiceId;

@Column(name = "origin_work_item_id")
public UUID originWorkItemId;

@Column(name = "origin_version")
public Long originVersion;
```

- [ ] **Step 7: Update WorkItemEntityMapper**

Map `originServiceId`, `originWorkItemId`, and `originVersion` in both directions. In `copyFieldsToEntity()`, explicitly include `originVersion` but continue to exclude `version` (JPA `@Version`).

- [ ] **Step 8: Override findByOrigin() in JPA store**

```java
// In JpaWorkItemStore:
@Override
public Optional<WorkItem> findByOrigin(String originServiceId, UUID originWorkItemId) {
    return find("originServiceId = ?1 and originWorkItemId = ?2",
                originServiceId, originWorkItemId)
        .firstResultOptional()
        .map(WorkItemEntityMapper::toWorkItem);
}
```

- [ ] **Step 9: Override findByOrigin() in InMemory and MongoDB stores**

Follow the same pattern. InMemory: stream filter. MongoDB: document query.

- [ ] **Step 10: Write Flyway migration**

```sql
-- V40__federation_fields.sql
ALTER TABLE work_item ADD COLUMN origin_service_id VARCHAR(255);
ALTER TABLE work_item ADD COLUMN origin_work_item_id UUID;
ALTER TABLE work_item ADD COLUMN origin_version BIGINT;
CREATE INDEX idx_work_item_origin ON work_item (origin_service_id, origin_work_item_id);
```

- [ ] **Step 11: Write mapper test**

```java
// runtime/src/test/java/.../WorkItemEntityMapperFederationTest.java
@Test
void mapsFederationFieldsBidirectionally() {
    var workItem = WorkItem.builder()
        .id(UUID.randomUUID()).title("test").createdBy("sys")
        .status(WorkItemStatus.PENDING).priority(WorkItemPriority.MEDIUM)
        .originServiceId("svc-a").originWorkItemId(UUID.randomUUID()).originVersion(3L)
        .build();
    var entity = WorkItemEntityMapper.toEntity(workItem);
    assertEquals("svc-a", entity.originServiceId);
    var roundTripped = WorkItemEntityMapper.toWorkItem(entity);
    assertEquals("svc-a", roundTripped.originServiceId());
    assertEquals(3L, roundTripped.originVersion());
}

@Test
void copyFieldsToEntityExcludesJpaVersionButIncludesOriginVersion() {
    var entity = new WorkItemEntity();
    entity.id = UUID.randomUUID();
    entity.version = 10L; // JPA-managed
    var workItem = WorkItem.builder()
        .id(entity.id).title("test").createdBy("sys")
        .status(WorkItemStatus.PENDING).priority(WorkItemPriority.MEDIUM)
        .version(99L) // should NOT overwrite entity.version
        .originVersion(5L) // SHOULD be copied
        .build();
    WorkItemEntityMapper.copyFieldsToEntity(workItem, entity);
    assertEquals(10L, entity.version); // JPA version preserved
    assertEquals(5L, entity.originVersion); // origin version copied
}
```

- [ ] **Step 12: Run all affected module tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
```

- [ ] **Step 13: Commit**

```bash
git add api/ runtime/ persistence-memory/ persistence-mongodb/
git commit -m "feat(#95): add federation fields to WorkItem record and store backends

Add originServiceId, originWorkItemId, originVersion to WorkItem record.
Add findByOrigin() to WorkItemStore SPI with JPA/MongoDB/InMemory overrides.
V40 migration adds columns and composite index.

Refs #95 Refs #92"
```

### Task 2: FederationGuardStore decorator and FederationSyncContext

**Files:**
- Create: `federation/pom.xml` — new Maven module
- Create: `federation/src/main/java/io/casehub/work/federation/FederationSyncContext.java`
- Create: `federation/src/main/java/io/casehub/work/federation/FederationGuardStore.java`
- Create: `federation/src/main/java/io/casehub/work/federation/FederatedWorkItemMutationException.java`
- Test: `federation/src/test/java/io/casehub/work/federation/FederationGuardStoreTest.java`

**Interfaces:**
- Consumes: `WorkItemStore` SPI (api/), `WorkItem.originServiceId()`
- Produces: `FederationSyncContext.activate()/.deactivate()/.isActive()`, `FederationGuardStore` CDI decorator

- [ ] **Step 1: Create federation/ module scaffold**

Create `federation/pom.xml` with dependencies on `casehub-work-api` and `casehub-work` (runtime). Add the module to the parent `pom.xml` `<modules>` list. Include `jandex-maven-plugin` for CDI bean discovery.

- [ ] **Step 2: Write failing test**

```java
@QuarkusComponentTest({FederationGuardStore.class, FederationSyncContext.class})
class FederationGuardStoreTest {

    @InjectMock
    WorkItemStore delegate;

    @Inject
    WorkItemStore guardedStore; // CDI injects the decorator

    @Test
    void rejectsShadowMutationWithoutSyncContext() {
        var shadow = WorkItem.builder()
            .id(UUID.randomUUID()).title("shadow").createdBy("sys")
            .status(WorkItemStatus.ASSIGNED).priority(WorkItemPriority.MEDIUM)
            .originServiceId("remote-svc")
            .originWorkItemId(UUID.randomUUID())
            .build();
        assertThrows(FederatedWorkItemMutationException.class,
            () -> guardedStore.put(shadow));
        verify(delegate, never()).put(any());
    }

    @Test
    void allowsShadowMutationWithSyncContext() {
        var shadow = WorkItem.builder()
            .id(UUID.randomUUID()).title("shadow").createdBy("sys")
            .status(WorkItemStatus.ASSIGNED).priority(WorkItemPriority.MEDIUM)
            .originServiceId("remote-svc")
            .originWorkItemId(UUID.randomUUID())
            .build();
        when(delegate.put(shadow)).thenReturn(shadow);
        FederationSyncContext.activate();
        try {
            guardedStore.put(shadow);
            verify(delegate).put(shadow);
        } finally {
            FederationSyncContext.deactivate();
        }
    }

    @Test
    void allowsLocalWorkItemMutation() {
        var local = WorkItem.builder()
            .id(UUID.randomUUID()).title("local").createdBy("sys")
            .status(WorkItemStatus.PENDING).priority(WorkItemPriority.MEDIUM)
            .build();
        when(delegate.put(local)).thenReturn(local);
        guardedStore.put(local);
        verify(delegate).put(local);
    }
}
```

- [ ] **Step 3: Run test — verify FAIL**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FederationGuardStoreTest -pl federation
```

- [ ] **Step 4: Implement FederationSyncContext**

```java
public final class FederationSyncContext {
    private static final ThreadLocal<Boolean> ACTIVE = ThreadLocal.withInitial(() -> false);
    public static void activate() { ACTIVE.set(true); }
    public static void deactivate() { ACTIVE.remove(); }
    public static boolean isActive() { return ACTIVE.get(); }
    private FederationSyncContext() {}
}
```

- [ ] **Step 5: Implement FederationGuardStore**

```java
@Decorator
@Priority(Interceptor.Priority.APPLICATION)
@ApplicationScoped
public class FederationGuardStore implements WorkItemStore {

    @Delegate @Inject @Any
    WorkItemStore delegate;

    @Override
    public WorkItem put(WorkItem item) {
        if (item.originServiceId() != null && !FederationSyncContext.isActive()) {
            throw new FederatedWorkItemMutationException(item.id());
        }
        return delegate.put(item);
    }

    // All other methods delegate transparently via @Delegate
}
```

- [ ] **Step 6: Run test — verify PASS**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -Dtest=FederationGuardStoreTest -pl federation
```

- [ ] **Step 7: Commit**

```bash
git add federation/ pom.xml
git commit -m "feat(#95): add FederationGuardStore decorator and FederationSyncContext

CDI @Decorator on WorkItemStore.put() rejects shadow WorkItem mutations
unless FederationSyncContext is active. Single persistence-boundary guard
catches all mutation paths.

Refs #95 Refs #92"
```

### Task 3: Extract WorkItemService interface

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java` — interface with public lifecycle + query methods
- Modify: `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java` — `implements WorkItemOperations`
- Modify: all injection sites referencing `WorkItemService` → inject `WorkItemOperations` interface
- Test: compile + existing tests pass

**Interfaces:**
- Consumes: existing `WorkItemService` public methods
- Produces: `WorkItemOperations` interface in `api/` — used by `FederationProxyService` decorator in Batch 4

- [ ] **Step 1: Identify all public methods on WorkItemService**

Use `ide_file_structure` on `runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java`. Extract method signatures for lifecycle operations (create, claim, start, complete, reject, delegate, release, suspend, resume, cancel, fault, obsolete, escalate, extend, updateDeadline, addLabel, removeLabel, clone) and query operations (findById, scan, findByCallerRef, findActiveByCallerRef, findChildrenByParentId).

- [ ] **Step 2: Create WorkItemOperations interface in api/**

```java
// api/src/main/java/io/casehub/work/api/spi/WorkItemOperations.java
package io.casehub.work.api.spi;

public interface WorkItemOperations {
    WorkItem create(WorkItemCreateRequest request);
    WorkItem claim(UUID id, String claimantId);
    WorkItem start(UUID id, String actorId);
    WorkItem complete(UUID id, String actorId, String resolution, String outcome);
    // ... all public lifecycle + query methods
    Optional<WorkItem> findById(UUID id);
    List<WorkItem> scan(WorkItemQuery query);
}
```

- [ ] **Step 3: Make WorkItemService implement the interface**

Use `ide_edit_member` to add `implements WorkItemOperations` to the class declaration. All methods already have the right signatures — no implementation changes needed.

- [ ] **Step 4: Use ide_find_references to find all WorkItemService injection sites**

```
ide_find_references(symbol="io.casehub.work.runtime.service.WorkItemService")
```

Update injection sites in REST resources, adapters, and other consumers to inject `WorkItemOperations` instead of the concrete class. Use `ide_refactor_rename` if appropriate.

- [ ] **Step 5: Run full test suite for affected modules**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl rest
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl engine-adapter
```

- [ ] **Step 6: Commit**

```bash
git add api/ runtime/ rest/ engine-adapter/ qhorus/
git commit -m "refactor(#95): extract WorkItemOperations interface from WorkItemService

Mechanical extraction — all public lifecycle and query methods moved to
WorkItemOperations interface in api/. WorkItemService implements it.
Injection sites updated. Enables CDI @Decorator for federation proxy.

Refs #95 Refs #92"
```

---

## Batch 2: Protocol — Transport SPI + Subscriptions

After this batch: federation transport is pluggable with a webhook implementation, and services can register subscriptions with filter predicates.

### Task 4: FederationTransport SPI + webhook implementation

**Files:**
- Create: `api/src/main/java/io/casehub/work/api/spi/FederationTransport.java` — SPI interface
- Create: `federation/src/main/java/io/casehub/work/federation/transport/WebhookFederationTransport.java`
- Create: `federation/src/main/java/io/casehub/work/federation/transport/HmacSigner.java`
- Create: `federation/src/main/java/io/casehub/work/federation/transport/HmacVerifier.java`
- Test: `federation/src/test/java/io/casehub/work/federation/transport/HmacSignerTest.java`
- Test: `federation/src/test/java/io/casehub/work/federation/transport/WebhookFederationTransportTest.java`

**Interfaces:**
- Consumes: CloudEvents SDK (`io.cloudevents:cloudevents-core`)
- Produces: `FederationTransport.send(CloudEvent, callbackUrl, hmacSecret)`, `HmacVerifier.verify(payload, signature, secret)`

- [ ] **Step 1: Write HmacSigner test**

```java
@Test
void signAndVerifyRoundTrips() {
    byte[] secret = "test-secret".getBytes(UTF_8);
    String payload = "{\"type\":\"io.casehub.work.federation.created\"}";
    String signature = HmacSigner.sign(payload, secret);
    assertTrue(HmacVerifier.verify(payload, signature, secret));
}

@Test
void rejectsWrongSignature() {
    byte[] secret = "test-secret".getBytes(UTF_8);
    assertFalse(HmacVerifier.verify("payload", "wrong-sig", secret));
}
```

- [ ] **Step 2: Implement HmacSigner/Verifier**

HMAC-SHA256 using `javax.crypto.Mac`. Constant-time comparison in verifier (`MessageDigest.isEqual`).

- [ ] **Step 3: Define FederationTransport SPI**

```java
public interface FederationTransport {
    void send(CloudEvent event, String callbackUrl, byte[] hmacSecret);
}
```

- [ ] **Step 4: Implement WebhookFederationTransport**

HTTP POST with `Content-Type: application/cloudevents+json`, HMAC signature in `X-Federation-Signature` header. Use `java.net.http.HttpClient` (no external dependencies). Configurable timeout.

- [ ] **Step 5: Run tests, commit**

### Task 5: Subscription model

**Files:**
- Create: `federation/src/main/java/io/casehub/work/federation/subscription/FederationSubscriptionEntity.java`
- Create: `federation/src/main/java/io/casehub/work/federation/subscription/FederationSubscriptionTrackingEntity.java`
- Create: `federation/src/main/java/io/casehub/work/federation/subscription/FederationSubscriptionStore.java`
- Create: `federation/src/main/java/io/casehub/work/federation/subscription/FederationSubscriptionService.java`
- Create: `federation/src/main/java/io/casehub/work/federation/subscription/SubscriptionFilter.java`
- Create: `federation/src/main/java/io/casehub/work/federation/subscription/SubscriptionFilterEvaluator.java`
- Create: `federation/src/main/java/io/casehub/work/federation/rest/FederationSubscriptionResource.java`
- Create: `federation/src/main/resources/db/work/migration/V8000__federation_subscription.sql`
- Test: `federation/src/test/java/io/casehub/work/federation/subscription/SubscriptionFilterEvaluatorTest.java`
- Test: `federation/src/test/java/io/casehub/work/federation/subscription/FederationSubscriptionServiceTest.java`

**Interfaces:**
- Consumes: `WorkItem.candidateGroups()`, `WorkItem.candidateUsers()`, `WorkItem.tenancyId()`
- Produces: `FederationSubscriptionService.register()`, `.matchSubscriptions(WorkItem)`, `.lockOn(subscriptionId, workItemId)`, `.findLockedSubscriptions(workItemId)`

- [ ] **Step 1: Write filter evaluator test**

```java
@Test
void matchesCandidateGroupIntersection() {
    var filter = new SubscriptionFilter(List.of("legal", "compliance"), List.of(), "tenant-1");
    var item = WorkItem.builder()
        .id(UUID.randomUUID()).title("test").createdBy("sys")
        .status(WorkItemStatus.PENDING).priority(WorkItemPriority.MEDIUM)
        .candidateGroups("legal,finance").tenancyId("tenant-1")
        .build();
    assertTrue(SubscriptionFilterEvaluator.matches(filter, item));
}

@Test
void rejectsTenancyMismatch() {
    var filter = new SubscriptionFilter(List.of("legal"), List.of(), "tenant-1");
    var item = WorkItem.builder()
        .id(UUID.randomUUID()).title("test").createdBy("sys")
        .status(WorkItemStatus.PENDING).priority(WorkItemPriority.MEDIUM)
        .candidateGroups("legal").tenancyId("tenant-2")
        .build();
    assertFalse(SubscriptionFilterEvaluator.matches(filter, item));
}
```

- [ ] **Step 2: Implement filter evaluator, entities, store, service, REST endpoint, migration**

Follow the spec §8 for filter semantics (OR combination, empty arrays ignored, tenant mandatory). JPA entities with Panache. REST resource at `/federation/subscriptions`.

Migration V8000:
```sql
CREATE TABLE federation_subscription (
    id UUID PRIMARY KEY,
    peer_id VARCHAR(255) NOT NULL,
    callback_url VARCHAR(1024) NOT NULL,
    tenancy_id VARCHAR(255) NOT NULL,
    filter_json TEXT NOT NULL,
    capabilities_json TEXT,
    hmac_secret_hash VARCHAR(255) NOT NULL,
    status VARCHAR(32) NOT NULL DEFAULT 'ACTIVE',
    consecutive_failures INT NOT NULL DEFAULT 0,
    created_at TIMESTAMP NOT NULL
);

CREATE TABLE federation_subscription_tracking (
    subscription_id UUID NOT NULL REFERENCES federation_subscription(id),
    work_item_id UUID NOT NULL,
    PRIMARY KEY (subscription_id, work_item_id)
);
```

- [ ] **Step 3: Run tests, commit**

---

## Batch 3: Federation Engine — Receiver + Router

After this batch: inbound CloudEvents create/update shadow WorkItems, and outbound lifecycle events are routed to matching subscriptions.

### Task 6: FederationReceiver — inbound CloudEvents to shadow WorkItems

**Files:**
- Create: `federation/src/main/java/io/casehub/work/federation/FederationReceiver.java`
- Create: `federation/src/main/java/io/casehub/work/federation/rest/FederationEventResource.java`
- Test: `federation/src/test/java/io/casehub/work/federation/FederationReceiverTest.java`

**Interfaces:**
- Consumes: `WorkItemStore.put()`, `WorkItemStore.findByOrigin()`, `FederationSyncContext`, `HmacVerifier`, `TenantContextRunner`, `WorkItemLifecycleEmitter`
- Produces: `FederationReceiver.onEvent(CloudEvent)` — creates/updates shadow WorkItems

- [ ] **Step 1: Write test for shadow creation from CloudEvent**

Test that receiving a `io.casehub.work.federation.created` CloudEvent creates a shadow WorkItem with correct `originServiceId` and `originWorkItemId`. Verify `FederationSyncContext` is activated during `put()`.

- [ ] **Step 2: Write test for stale event rejection**

Test that a CloudEvent with `workitemversion <= existing shadow.originVersion()` is discarded.

- [ ] **Step 3: Write test for callerRef namespacing**

Test that the incoming `callerRef` is prefixed with `federation:<originServiceId>:` to prevent collisions.

- [ ] **Step 4: Implement FederationReceiver**

Follow spec §10 — all 12 steps including tenant context, version check, callerRef namespacing, SyncContext activation, store put, audit, lifecycle emission.

REST endpoint at `POST /federation/events` — receives CloudEvents, verifies HMAC, delegates to receiver.

- [ ] **Step 5: Run tests, commit**

### Task 7: FederationEventRouter — outbound lifecycle events to subscriptions

**Files:**
- Create: `federation/src/main/java/io/casehub/work/federation/FederationEventRouter.java`
- Create: `federation/src/main/java/io/casehub/work/federation/FederationCloudEventBuilder.java`
- Test: `federation/src/test/java/io/casehub/work/federation/FederationEventRouterTest.java`

**Interfaces:**
- Consumes: `WorkItemLifecycleEvent`, `FederationSubscriptionService`, `FederationTransport`
- Produces: outbound CloudEvents to all matching subscriptions

- [ ] **Step 1: Write test for feedback loop prevention**

Test that events from shadow WorkItems (`originServiceId != null`) are skipped.

- [ ] **Step 2: Write test for subscription matching on creation**

Test that a `CREATED` event evaluates filters and locks on matching subscriptions.

- [ ] **Step 3: Write test for locked subscription delivery**

Test that subsequent events for a locked WorkItem are delivered to all locked subscriptions.

- [ ] **Step 4: Implement FederationEventRouter**

`@ObservesAsync WorkItemLifecycleEvent`. Check `originServiceId != null` → skip. For CREATED events: evaluate subscriptions, lock on matches. For all events: find locked subscriptions, build CloudEvents, send via transport.

- [ ] **Step 5: Run tests, commit**

---

## Batch 4: Client + Proxy

After this batch: a lightweight client module enables remote WorkItem operations, and the federation proxy decorator intercepts shadow mutations and routes them to the owner.

### Task 8: Lightweight client module

**Files:**
- Create: `client/pom.xml` — minimal dependencies (no JPA, no CDI, no Quarkus)
- Create: `client/src/main/java/io/casehub/work/client/WorkItemClient.java`
- Create: `client/src/main/java/io/casehub/work/client/WorkItemClientConfig.java`
- Test: `client/src/test/java/io/casehub/work/client/WorkItemClientTest.java`

**Interfaces:**
- Consumes: owner's REST API
- Produces: `WorkItemClient.claim(baseUrl, workItemId, claimantId, bearerToken)`, `.complete(...)`, `.reject(...)`, `.delegate(...)`, `.query(...)`

- [ ] **Step 1: Create module, write test with mock HTTP server**

Use `com.sun.net.httpserver.HttpServer` (JDK built-in) for test. Verify claim sends correct REST call, handles 200/409/503.

- [ ] **Step 2: Implement WorkItemClient**

`java.net.http.HttpClient` with configurable timeout. JSON serialization with Jackson (already a transitive dependency). No framework dependencies.

- [ ] **Step 3: Run tests, commit**

### Task 9: FederationProxyService decorator + audit

**Files:**
- Create: `federation/src/main/java/io/casehub/work/federation/FederationProxy.java` — wraps WorkItemClient with subscription/config lookup
- Create: `federation/src/main/java/io/casehub/work/federation/FederationProxyService.java` — CDI @Decorator on WorkItemOperations
- Test: `federation/src/test/java/io/casehub/work/federation/FederationProxyServiceTest.java`

**Interfaces:**
- Consumes: `WorkItemOperations` (Task 3), `WorkItemClient` (Task 8), `WorkItemStore.findById()`, `AuditEntryStore`
- Produces: transparent proxy — lifecycle operations on shadows route to owner, local operations pass through

- [ ] **Step 1: Write test for proxy intercept**

```java
@Test
void claimOnShadowProxiesToOwner() {
    // shadow WorkItem with originServiceId set
    // verify FederationProxy.claim() is called instead of delegate.claim()
}

@Test
void claimOnLocalDelegatesToService() {
    // local WorkItem with originServiceId null
    // verify delegate.claim() is called directly
}
```

- [ ] **Step 2: Implement FederationProxyService decorator**

CDI `@Decorator` on `WorkItemOperations`. Pattern: `findById()` → check `originServiceId` → proxy or delegate. Audit proxied operations locally with federation metadata in `detail` JSON.

- [ ] **Step 3: Run tests, commit**

---

## Batch 5: Integration

After this batch: end-to-end federation works — WorkItem created in one service appears in another, is claimable and completable across services.

### Task 10: End-to-end federation integration test

**Files:**
- Create: `federation/src/test/java/io/casehub/work/federation/FederationIntegrationTest.java`
- Create: `federation/src/test/java/io/casehub/work/federation/InMemoryFederationTransport.java` — test transport

**Interfaces:**
- Consumes: all federation components
- Produces: verified end-to-end flow

- [ ] **Step 1: Create InMemoryFederationTransport**

A `FederationTransport` implementation that synchronously delivers CloudEvents to a `FederationReceiver` in the same JVM. No HTTP, no network — pure in-memory for test speed.

- [ ] **Step 2: Write integration test**

```java
@QuarkusTest
class FederationIntegrationTest {

    @Test
    void createOnOwnerAppearsAsShadowOnConsumer() {
        // 1. Register subscription (filter: candidateGroups contains "test-group")
        // 2. Create WorkItem with candidateGroups="test-group"
        // 3. Verify shadow exists via findByOrigin()
        // 4. Verify shadow has correct originServiceId, originWorkItemId
        // 5. Verify shadow status matches owner
    }

    @Test
    void claimOnShadowUpdatesOwner() {
        // 1. Create WorkItem + shadow (as above)
        // 2. Claim shadow via WorkItemOperations.claim()
        // 3. Verify proxy called owner's REST API
        // 4. Verify shadow updated to ASSIGNED after CloudEvent delivery
    }

    @Test
    void guardPreventsDirectShadowMutation() {
        // 1. Create shadow
        // 2. Attempt store.put(shadow.toBuilder().status(COMPLETED).build())
        // 3. Verify FederatedWorkItemMutationException thrown
    }

    @Test
    void staleEventDiscarded() {
        // 1. Create shadow with originVersion=5
        // 2. Send event with workitemversion=3
        // 3. Verify shadow unchanged
    }
}
```

- [ ] **Step 3: Run integration tests**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn verify -pl federation
```

- [ ] **Step 4: Run full project test suite to check for regressions**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api,runtime,rest,engine-adapter,qhorus,federation
```

- [ ] **Step 5: Commit**

```bash
git add federation/
git commit -m "feat(#95): end-to-end federation integration test

Verifies: shadow creation, proxy claim, guard protection, stale event
rejection, subscription matching. Uses InMemoryFederationTransport.

Refs #95 Refs #92"
```

---

## References

- [2026-08-18-cross-service-federation-design.md] — design spec this plan implements
- [decisions.md] — 12 validated design decisions (D1–D12)
- api/src/main/java/io/casehub/work/api/WorkItem.java — WorkItem record (47 fields)
- api/src/main/java/io/casehub/work/api/spi/WorkItemStore.java — persistence SPI
- runtime/src/main/java/io/casehub/work/runtime/service/WorkItemService.java — lifecycle service
- runtime/src/main/java/io/casehub/work/runtime/event/WorkCloudEventAdapter.java — existing CloudEvents
- runtime/src/main/java/io/casehub/work/runtime/event/WorkItemLifecycleEmitter.java — event emission
- docs/FLYWAY.md — migration version ranges
- docs/GOTCHAS.md — CDI, JPA, testing gotchas
- docs/MODULES.md — module ownership
- GE-20260805-a2aa1b — WorkItemStore SPI co-location issue
- GE-20260521-87daa0 — @ObservesAsync in external JARs
- GE-20260421-cd3f95 — CDI observer recursive re-entry
- GitHub #95, #92
