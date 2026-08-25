# Worker Rights Generalization Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #221 — feat: worker rights model and authorization service SPI
**Issue group:** #221

**Goal:** Generalize engine-flavored worker rights SPI types in platform-api so any domain can use the worker rights infrastructure without depending on engine concepts.

**Architecture:** Replace closed `WorkerAction` enum with an open record, introduce `ResourceId` value type replacing `UUID caseId`, add `WorkerAuthorizationContext` marker interface for pluggable authorization contexts, and extract a reusable `WorkerCredentialFilter` into a new `acl-worker/` module with `WorkerScopeExtractor` SPI.

**Tech Stack:** Java 21 records, Quarkus CDI (Arc), Jakarta REST (JAX-RS ContainerRequestFilter), JUnit 5

## Global Constraints

- `platform-api/` must remain zero-dependency — no Quarkus, no JPA, no casehubio imports. Pure Java only.
- Every SPI in platform-api gets a @DefaultBean implementation in platform/.
- New `acl-worker/` module follows `acl-inmem/` pattern: compile dep on `casehub-platform-api`, `quarkus-arc`, and adds `jakarta.ws.rs-api`.
- Pre-release platform — breaking changes are expected and correct.
- IntelliJ MCP is required for all source file operations. Use `mcp__intellij-index__*` tools.

---

### Task 1: Foundation types — ResourceId and WorkerAction record

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/acl/ResourceId.java`
- Create: `platform-api/src/test/java/io/casehub/platform/api/acl/ResourceIdTest.java`
- Modify: `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAction.java`
- Modify: `platform-api/src/test/java/io/casehub/platform/api/acl/WorkerActionTest.java`

**Interfaces:**
- Consumes: `AclAction` enum (existing, unchanged)
- Produces: `ResourceId(String type, String id)` record, `WorkerAction(String name, AclAction aclAction)` record

- [ ] **Step 1: Write ResourceId tests**

```java
package io.casehub.platform.api.acl;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

class ResourceIdTest {

    @Test
    void constructsWithTypeAndId() {
        var rid = new ResourceId("case", "abc-123");
        assertEquals("case", rid.type());
        assertEquals("abc-123", rid.id());
    }

    @Test
    void toStringFormatsAsTypeColonId() {
        assertEquals("case:abc-123", new ResourceId("case", "abc-123").toString());
    }

    @Test
    void parseRoundTrips() {
        var rid = ResourceId.parse("case:abc-123");
        assertEquals("case", rid.type());
        assertEquals("abc-123", rid.id());
    }

    @Test
    void parseHandlesColonInId() {
        var rid = ResourceId.parse("ns:sub:value");
        assertEquals("ns", rid.type());
        assertEquals("sub:value", rid.id());
    }

    @Test
    void rejectsBlankType() {
        assertThrows(IllegalArgumentException.class, () -> new ResourceId("", "id"));
    }

    @Test
    void rejectsNullType() {
        assertThrows(IllegalArgumentException.class, () -> new ResourceId(null, "id"));
    }

    @Test
    void rejectsBlankId() {
        assertThrows(IllegalArgumentException.class, () -> new ResourceId("case", ""));
    }

    @Test
    void rejectsNullId() {
        assertThrows(IllegalArgumentException.class, () -> new ResourceId("case", null));
    }

    @Test
    void rejectsColonInType() {
        assertThrows(IllegalArgumentException.class, () -> new ResourceId("a:b", "id"));
    }

    @Test
    void parseRejectsNoColon() {
        assertThrows(IllegalArgumentException.class, () -> ResourceId.parse("nocolon"));
    }

    @Test
    void parseRejectsLeadingColon() {
        assertThrows(IllegalArgumentException.class, () -> ResourceId.parse(":id"));
    }

    @Test
    void parseRejectsTrailingColon() {
        assertThrows(IllegalArgumentException.class, () -> ResourceId.parse("type:"));
    }

    @Test
    void equalityByValue() {
        assertEquals(new ResourceId("case", "123"), new ResourceId("case", "123"));
        assertNotEquals(new ResourceId("case", "123"), new ResourceId("case", "456"));
        assertNotEquals(new ResourceId("case", "123"), new ResourceId("plan", "123"));
    }
}
```

- [ ] **Step 2: Run ResourceId tests — verify they fail**

Run: `mvn --batch-mode -pl platform-api test -Dtest=ResourceIdTest`
Expected: FAIL — `ResourceId` class does not exist

- [ ] **Step 3: Implement ResourceId**

Create `platform-api/src/main/java/io/casehub/platform/api/acl/ResourceId.java`:

```java
package io.casehub.platform.api.acl;

public record ResourceId(String type, String id) {

    public ResourceId {
        if (type == null || type.isBlank()) {
            throw new IllegalArgumentException("type must not be blank");
        }
        if (type.contains(":")) {
            throw new IllegalArgumentException("type must not contain ':'");
        }
        if (id == null || id.isBlank()) {
            throw new IllegalArgumentException("id must not be blank");
        }
    }

    @Override
    public String toString() {
        return type + ":" + id;
    }

    public static ResourceId parse(String value) {
        if (value == null) {
            throw new IllegalArgumentException("value must not be null");
        }
        int colon = value.indexOf(':');
        if (colon <= 0 || colon == value.length() - 1) {
            throw new IllegalArgumentException(
                "ResourceId must be 'type:id', got: " + value);
        }
        return new ResourceId(value.substring(0, colon), value.substring(colon + 1));
    }
}
```

- [ ] **Step 4: Run ResourceId tests — verify they pass**

Run: `mvn --batch-mode -pl platform-api test -Dtest=ResourceIdTest`
Expected: PASS

- [ ] **Step 5: Write WorkerAction record tests**

Replace `platform-api/src/test/java/io/casehub/platform/api/acl/WorkerActionTest.java`:

```java
package io.casehub.platform.api.acl;

import static org.junit.jupiter.api.Assertions.*;
import org.junit.jupiter.api.Test;

class WorkerActionTest {

    @Test
    void constructsWithNameAndAclAction() {
        var action = new WorkerAction("READ_CONTEXT", AclAction.READ);
        assertEquals("READ_CONTEXT", action.name());
        assertEquals(AclAction.READ, action.aclAction());
    }

    @Test
    void equalityByValue() {
        assertEquals(
            new WorkerAction("READ_CONTEXT", AclAction.READ),
            new WorkerAction("READ_CONTEXT", AclAction.READ));
    }

    @Test
    void differentNameNotEqual() {
        assertNotEquals(
            new WorkerAction("READ_CONTEXT", AclAction.READ),
            new WorkerAction("WRITE_CONTEXT", AclAction.READ));
    }

    @Test
    void differentAclActionNotEqual() {
        assertNotEquals(
            new WorkerAction("READ_CONTEXT", AclAction.READ),
            new WorkerAction("READ_CONTEXT", AclAction.WRITE));
    }

    @Test
    void rejectsBlankName() {
        assertThrows(IllegalArgumentException.class,
            () -> new WorkerAction("", AclAction.READ));
    }

    @Test
    void rejectsNullName() {
        assertThrows(IllegalArgumentException.class,
            () -> new WorkerAction(null, AclAction.READ));
    }

    @Test
    void rejectsNullAclAction() {
        assertThrows(IllegalArgumentException.class,
            () -> new WorkerAction("READ_CONTEXT", null));
    }
}
```

- [ ] **Step 6: Run WorkerAction tests — verify they fail**

Run: `mvn --batch-mode -pl platform-api test -Dtest=WorkerActionTest`
Expected: FAIL — `WorkerAction` is still an enum

- [ ] **Step 7: Replace WorkerAction enum with record**

Replace `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAction.java`:

```java
package io.casehub.platform.api.acl;

public record WorkerAction(String name, AclAction aclAction) {

    public WorkerAction {
        if (name == null || name.isBlank()) {
            throw new IllegalArgumentException("name must not be blank");
        }
        if (aclAction == null) {
            throw new IllegalArgumentException("aclAction must not be null");
        }
    }
}
```

- [ ] **Step 8: Run WorkerAction tests — verify they pass**

Run: `mvn --batch-mode -pl platform-api test -Dtest=WorkerActionTest`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/114/platform add platform-api/src/main/java/io/casehub/platform/api/acl/ResourceId.java platform-api/src/test/java/io/casehub/platform/api/acl/ResourceIdTest.java platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAction.java platform-api/src/test/java/io/casehub/platform/api/acl/WorkerActionTest.java
git -C /Users/mdproctor/claude/casehub/slots/114/platform commit -m "feat(#221): add ResourceId value type, convert WorkerAction enum to record"
```

---

### Task 2: SPI generalization — WorkerCredential, WorkerCredentialStore, WorkerPermissionRequest, WorkerAuthorizationContext

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAuthorizationContext.java`
- Modify: `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerCredential.java`
- Modify: `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerCredentialStore.java`
- Modify: `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerPermissionRequest.java`
- Modify: `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAuthorizationDeniedException.java`
- Create: `platform-api/src/test/java/io/casehub/platform/api/acl/WorkerCredentialTest.java`
- Create: `platform-api/src/test/java/io/casehub/platform/api/acl/WorkerPermissionRequestTest.java`

**Interfaces:**
- Consumes: `ResourceId`, `WorkerAction` record (from Task 1)
- Produces: `WorkerAuthorizationContext` interface, generalized `WorkerCredential(String token, String actorId, ResourceId resourceId, String tenancyId, Set<WorkerAction> actions, Instant expiresAt, Instant createdAt)`, generalized `WorkerCredentialStore` with `revokeByResource(ResourceId)` and `findActiveByActorAndResource(String, ResourceId)`, generalized `WorkerPermissionRequest(String actorId, String resourceType, Set<WorkerAction> actions, WorkerAuthorizationContext context, String tenancyId)`

- [ ] **Step 1: Write WorkerCredential tests with ResourceId**

```java
package io.casehub.platform.api.acl;

import static org.junit.jupiter.api.Assertions.*;
import java.time.Instant;
import java.util.Set;
import org.junit.jupiter.api.Test;

class WorkerCredentialTest {

    private static final WorkerAction READ =
        new WorkerAction("READ_CONTEXT", AclAction.READ);

    @Test
    void constructsWithResourceId() {
        var rid = new ResourceId("case", "abc");
        var cred = new WorkerCredential("tok", "actor", rid, "tenant",
            Set.of(READ), Instant.now().plusSeconds(60), Instant.now());
        assertEquals(rid, cred.resourceId());
    }

    @Test
    void isExpired_returnsTrueWhenPastExpiry() {
        var cred = new WorkerCredential("tok", "actor",
            new ResourceId("case", "abc"), "tenant",
            Set.of(READ), Instant.now().minusSeconds(60), Instant.now().minusSeconds(120));
        assertTrue(cred.isExpired());
    }

    @Test
    void isExpired_returnsFalseWhenBeforeExpiry() {
        var cred = new WorkerCredential("tok", "actor",
            new ResourceId("case", "abc"), "tenant",
            Set.of(READ), Instant.now().plusSeconds(3600), Instant.now());
        assertFalse(cred.isExpired());
    }
}
```

- [ ] **Step 2: Write WorkerPermissionRequest test with WorkerAuthorizationContext**

```java
package io.casehub.platform.api.acl;

import static org.junit.jupiter.api.Assertions.*;
import java.util.Set;
import org.junit.jupiter.api.Test;

class WorkerPermissionRequestTest {

    record TestContext(String definitionId) implements WorkerAuthorizationContext {}

    @Test
    void constructsWithMarkerInterfaceContext() {
        var ctx = new TestContext("my-def");
        var req = new WorkerPermissionRequest("actor", "case",
            Set.of(new WorkerAction("READ", AclAction.READ)), ctx, "tenant");
        assertInstanceOf(TestContext.class, req.context());
        assertEquals("my-def", ((TestContext) req.context()).definitionId());
    }
}
```

- [ ] **Step 3: Run tests — verify they fail**

Run: `mvn --batch-mode -pl platform-api test -Dtest="WorkerCredentialTest,WorkerPermissionRequestTest"`
Expected: FAIL — old signatures don't match

- [ ] **Step 4: Create WorkerAuthorizationContext**

Create `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAuthorizationContext.java`:

```java
package io.casehub.platform.api.acl;

public interface WorkerAuthorizationContext {}
```

- [ ] **Step 5: Update WorkerCredential — caseId → resourceId**

Replace `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerCredential.java`:

```java
package io.casehub.platform.api.acl;

import java.time.Instant;
import java.util.Set;

public record WorkerCredential(
    String token,
    String actorId,
    ResourceId resourceId,
    String tenancyId,
    Set<WorkerAction> actions,
    Instant expiresAt,
    Instant createdAt) {

    public boolean isExpired() {
        return Instant.now().isAfter(expiresAt);
    }
}
```

- [ ] **Step 6: Update WorkerCredentialStore — method signatures**

Replace `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerCredentialStore.java`:

```java
package io.casehub.platform.api.acl;

import java.util.List;
import java.util.Optional;

public interface WorkerCredentialStore {

    default void store(WorkerCredential credential) {
    }

    default Optional<WorkerCredential> lookup(String token) {
        return Optional.empty();
    }

    default void revoke(String token) {
    }

    default List<WorkerCredential> revokeByResource(ResourceId resourceId) {
        return List.of();
    }

    default List<WorkerCredential> revokeByActor(String actorId) {
        return List.of();
    }

    default List<WorkerCredential> findActiveByActorAndResource(
        String actorId, ResourceId resourceId) {
        return List.of();
    }
}
```

- [ ] **Step 7: Update WorkerPermissionRequest — context generalization**

Replace `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerPermissionRequest.java`:

```java
package io.casehub.platform.api.acl;

import java.util.Set;

public record WorkerPermissionRequest(
    String actorId,
    String resourceType,
    Set<WorkerAction> actions,
    WorkerAuthorizationContext context,
    String tenancyId) {
}
```

- [ ] **Step 8: Update WorkerAuthorizationDeniedException — rename field**

Replace `platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAuthorizationDeniedException.java`:

```java
package io.casehub.platform.api.acl;

public class WorkerAuthorizationDeniedException extends SecurityException {

    private final String actorId;
    private final String definitionId;
    private final String reason;

    public WorkerAuthorizationDeniedException(String actorId, String definitionId, String reason) {
        super("Worker authorization denied: actor=" + actorId
              + " definition=" + definitionId + " reason=" + reason);
        this.actorId = actorId;
        this.definitionId = definitionId;
        this.reason = reason;
    }

    public String actorId() {
        return actorId;
    }

    public String definitionId() {
        return definitionId;
    }

    public String reason() {
        return reason;
    }
}
```

- [ ] **Step 9: Run tests — verify they pass**

Run: `mvn --batch-mode -pl platform-api test -Dtest="WorkerCredentialTest,WorkerPermissionRequestTest"`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/114/platform add platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAuthorizationContext.java platform-api/src/main/java/io/casehub/platform/api/acl/WorkerCredential.java platform-api/src/main/java/io/casehub/platform/api/acl/WorkerCredentialStore.java platform-api/src/main/java/io/casehub/platform/api/acl/WorkerPermissionRequest.java platform-api/src/main/java/io/casehub/platform/api/acl/WorkerAuthorizationDeniedException.java platform-api/src/test/java/io/casehub/platform/api/acl/WorkerCredentialTest.java platform-api/src/test/java/io/casehub/platform/api/acl/WorkerPermissionRequestTest.java
git -C /Users/mdproctor/claude/casehub/slots/114/platform commit -m "feat(#221): generalize WorkerCredential, WorkerCredentialStore, WorkerPermissionRequest — ResourceId + WorkerAuthorizationContext"
```

---

### Task 3: Implementation adaptation — NoOp and InMemory stores

**Files:**
- Modify: `platform/src/main/java/io/casehub/platform/acl/NoOpWorkerCredentialStore.java`
- Modify: `acl-inmem/src/main/java/io/casehub/platform/acl/inmem/InMemoryWorkerCredentialStore.java`
- Modify: `acl-inmem/src/test/java/io/casehub/platform/acl/inmem/InMemoryWorkerCredentialStoreTest.java`

**Interfaces:**
- Consumes: `WorkerCredentialStore` (generalized from Task 2), `ResourceId` (from Task 1), `WorkerAction` record (from Task 1)
- Produces: Working `NoOpWorkerCredentialStore` and `InMemoryWorkerCredentialStore` implementations

- [ ] **Step 1: Update InMemoryWorkerCredentialStoreTest**

Replace `acl-inmem/src/test/java/io/casehub/platform/acl/inmem/InMemoryWorkerCredentialStoreTest.java`:

```java
package io.casehub.platform.acl.inmem;

import static org.junit.jupiter.api.Assertions.assertEquals;
import static org.junit.jupiter.api.Assertions.assertTrue;

import io.casehub.platform.api.acl.AclAction;
import io.casehub.platform.api.acl.ResourceId;
import io.casehub.platform.api.acl.WorkerAction;
import io.casehub.platform.api.acl.WorkerCredential;
import java.time.Instant;
import java.util.Set;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class InMemoryWorkerCredentialStoreTest {

    private static final WorkerAction READ_CONTEXT =
        new WorkerAction("READ_CONTEXT", AclAction.READ);

    private InMemoryWorkerCredentialStore store;

    @BeforeEach
    void setUp() {
        store = new InMemoryWorkerCredentialStore();
    }

    @Test
    void storeAndLookup() {
        var cred = credential("t1", "actor-1", new ResourceId("case", "c1"), "tenant-1");
        store.store(cred);
        var result = store.lookup("t1");
        assertTrue(result.isPresent());
        assertEquals("actor-1", result.get().actorId());
    }

    @Test
    void lookup_unknownToken_returnsEmpty() {
        assertTrue(store.lookup("nonexistent").isEmpty());
    }

    @Test
    void revoke_removesCredential() {
        var cred = credential("t1", "actor-1", new ResourceId("case", "c1"), "tenant-1");
        store.store(cred);
        store.revoke("t1");
        assertTrue(store.lookup("t1").isEmpty());
    }

    @Test
    void revokeByResource_sweepsAllForResource() {
        var rid = new ResourceId("case", "c1");
        store.store(credential("t1", "actor-1", rid, "tenant-1"));
        store.store(credential("t2", "actor-2", rid, "tenant-1"));
        store.store(credential("t3", "actor-3", new ResourceId("case", "c2"), "tenant-1"));

        var revoked = store.revokeByResource(rid);

        assertEquals(2, revoked.size());
        assertTrue(store.lookup("t1").isEmpty());
        assertTrue(store.lookup("t2").isEmpty());
        assertTrue(store.lookup("t3").isPresent());
    }

    @Test
    void revokeByActor_sweepsAllForActor() {
        store.store(credential("t1", "agent:pool", new ResourceId("case", "c1"), "tenant-1"));
        store.store(credential("t2", "agent:pool", new ResourceId("case", "c2"), "tenant-1"));
        store.store(credential("t3", "agent:other", new ResourceId("case", "c3"), "tenant-1"));

        var revoked = store.revokeByActor("agent:pool");

        assertEquals(2, revoked.size());
        assertTrue(store.lookup("t1").isEmpty());
        assertTrue(store.lookup("t2").isEmpty());
        assertTrue(store.lookup("t3").isPresent());
    }

    @Test
    void findActiveByActorAndResource_excludesExpired() {
        var rid = new ResourceId("case", "c1");
        store.store(credential("t1", "agent:pool", rid, "tenant-1"));
        store.store(new WorkerCredential("t2", "agent:pool", rid, "tenant-1",
            Set.of(READ_CONTEXT), Instant.now().minusSeconds(60), Instant.now().minusSeconds(120)));

        var active = store.findActiveByActorAndResource("agent:pool", rid);

        assertEquals(1, active.size());
        assertEquals("t1", active.get(0).token());
    }

    @Test
    void lazyEviction_expiredRemovedOnStore() {
        store.store(new WorkerCredential("old", "actor-1", new ResourceId("case", "c1"), "tenant-1",
            Set.of(READ_CONTEXT), Instant.now().minusSeconds(60), Instant.now().minusSeconds(120)));
        store.store(credential("new", "actor-2", new ResourceId("case", "c2"), "tenant-1"));

        assertTrue(store.lookup("old").isEmpty());
        assertTrue(store.lookup("new").isPresent());
    }

    @Test
    void lazyEviction_expiredRemovedOnLookup() {
        store.store(new WorkerCredential("old", "actor-1", new ResourceId("case", "c1"), "tenant-1",
            Set.of(READ_CONTEXT), Instant.now().minusSeconds(60), Instant.now().minusSeconds(120)));

        store.lookup("anything");

        assertTrue(store.lookup("old").isEmpty());
    }

    private WorkerCredential credential(String token, String actorId, ResourceId resourceId, String tenancyId) {
        return new WorkerCredential(token, actorId, resourceId, tenancyId,
            Set.of(READ_CONTEXT), Instant.now().plusSeconds(3600), Instant.now());
    }
}
```

- [ ] **Step 2: Run tests — verify they fail**

Run: `mvn --batch-mode -pl acl-inmem test -Dtest=InMemoryWorkerCredentialStoreTest`
Expected: FAIL — `InMemoryWorkerCredentialStore` still has old method signatures

- [ ] **Step 3: Update InMemoryWorkerCredentialStore**

Replace `acl-inmem/src/main/java/io/casehub/platform/acl/inmem/InMemoryWorkerCredentialStore.java`:

```java
package io.casehub.platform.acl.inmem;

import io.casehub.platform.api.acl.ResourceId;
import io.casehub.platform.api.acl.WorkerCredential;
import io.casehub.platform.api.acl.WorkerCredentialStore;
import jakarta.annotation.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.util.List;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@Alternative
@Priority(10)
@ApplicationScoped
public class InMemoryWorkerCredentialStore implements WorkerCredentialStore {

    private final ConcurrentHashMap<String, WorkerCredential> store = new ConcurrentHashMap<>();

    @Override
    public void store(WorkerCredential credential) {
        evictExpired();
        store.put(credential.token(), credential);
    }

    @Override
    public Optional<WorkerCredential> lookup(String token) {
        evictExpired();
        return Optional.ofNullable(store.get(token));
    }

    @Override
    public void revoke(String token) {
        store.remove(token);
    }

    @Override
    public List<WorkerCredential> revokeByResource(ResourceId resourceId) {
        var revoked = store.values().stream()
            .filter(c -> c.resourceId().equals(resourceId)).toList();
        revoked.forEach(c -> store.remove(c.token()));
        return revoked;
    }

    @Override
    public List<WorkerCredential> revokeByActor(String actorId) {
        var revoked = store.values().stream()
            .filter(c -> c.actorId().equals(actorId)).toList();
        revoked.forEach(c -> store.remove(c.token()));
        return revoked;
    }

    @Override
    public List<WorkerCredential> findActiveByActorAndResource(String actorId, ResourceId resourceId) {
        return store.values().stream()
            .filter(c -> c.actorId().equals(actorId)
                && c.resourceId().equals(resourceId) && !c.isExpired())
            .toList();
    }

    private void evictExpired() {
        store.values().removeIf(WorkerCredential::isExpired);
    }
}
```

- [ ] **Step 4: NoOpWorkerCredentialStore — no changes needed**

The `NoOpWorkerCredentialStore` uses default methods from the interface. Since the
interface defaults were updated in Task 2, the NoOp already compiles correctly.
Verify: `NoOpWorkerCredentialStore implements WorkerCredentialStore` — all methods
default to no-op. No source changes required.

- [ ] **Step 5: Run tests — verify they pass**

Run: `mvn --batch-mode -pl acl-inmem test -Dtest=InMemoryWorkerCredentialStoreTest`
Expected: PASS

- [ ] **Step 6: Run full platform-api + acl-inmem build**

Run: `mvn --batch-mode -pl platform-api,platform,acl-inmem test`
Expected: PASS — all modules compile and test green

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/114/platform add acl-inmem/src/main/java/io/casehub/platform/acl/inmem/InMemoryWorkerCredentialStore.java acl-inmem/src/test/java/io/casehub/platform/acl/inmem/InMemoryWorkerCredentialStoreTest.java
git -C /Users/mdproctor/claude/casehub/slots/114/platform commit -m "feat(#221): adapt InMemoryWorkerCredentialStore to ResourceId and WorkerAction record"
```

---

### Task 4: New acl-worker module — filter + WorkerScopeExtractor SPI

**Files:**
- Create: `acl-worker/pom.xml`
- Create: `acl-worker/src/main/java/io/casehub/platform/acl/worker/WorkerScopeExtractor.java`
- Create: `acl-worker/src/main/java/io/casehub/platform/acl/worker/FailClosedWorkerScopeExtractor.java`
- Create: `acl-worker/src/main/java/io/casehub/platform/acl/worker/WorkerCredentialFilter.java`
- Create: `acl-worker/src/test/java/io/casehub/platform/acl/worker/WorkerCredentialFilterTest.java`
- Modify: `pom.xml` (root — add `acl-worker` module)

**Interfaces:**
- Consumes: `WorkerCredentialStore`, `WorkerCredential`, `ResourceId` (from Tasks 1-3), `CurrentPrincipal` (existing platform-api type)
- Produces: `WorkerScopeExtractor` SPI, `FailClosedWorkerScopeExtractor @DefaultBean`, `WorkerCredentialFilter @Provider`

- [ ] **Step 1: Create acl-worker/pom.xml**

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>

    <parent>
        <groupId>io.casehub</groupId>
        <artifactId>casehub-platform-parent</artifactId>
        <version>0.2-SNAPSHOT</version>
    </parent>

    <artifactId>casehub-platform-acl-worker</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform ACL Worker</name>
    <description>Reusable WorkerCredentialFilter for JAX-RS services that receive worker requests.
        Includes WorkerScopeExtractor SPI for domain-specific resource scope validation.
        Default: fail-closed (rejects all worker credentials unless a real ScopeExtractor is provided).
        Add as compile dep to activate. Do NOT add to services that don't receive worker requests.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <dependency>
            <groupId>jakarta.ws.rs</groupId>
            <artifactId>jakarta.ws.rs-api</artifactId>
        </dependency>
        <dependency>
            <groupId>org.jboss.logging</groupId>
            <artifactId>jboss-logging</artifactId>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
</project>
```

- [ ] **Step 2: Add acl-worker to root pom.xml modules**

Add `<module>acl-worker</module>` after `<module>acl-admin</module>` in the root `pom.xml` `<modules>` section.

- [ ] **Step 3: Create WorkerScopeExtractor SPI**

Create `acl-worker/src/main/java/io/casehub/platform/acl/worker/WorkerScopeExtractor.java`:

```java
package io.casehub.platform.acl.worker;

import io.casehub.platform.api.acl.ResourceId;
import jakarta.ws.rs.container.ContainerRequestContext;
import java.util.Optional;

public interface WorkerScopeExtractor {
    Optional<ResourceId> extractResourceId(ContainerRequestContext ctx);
}
```

- [ ] **Step 4: Create FailClosedWorkerScopeExtractor**

Create `acl-worker/src/main/java/io/casehub/platform/acl/worker/FailClosedWorkerScopeExtractor.java`:

```java
package io.casehub.platform.acl.worker;

import io.casehub.platform.api.acl.ResourceId;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.ws.rs.container.ContainerRequestContext;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class FailClosedWorkerScopeExtractor implements WorkerScopeExtractor {

    private static final ResourceId NEVER_MATCH =
        new ResourceId("__deny__", "__no_scope_extractor_configured__");

    @Override
    public Optional<ResourceId> extractResourceId(ContainerRequestContext ctx) {
        return Optional.of(NEVER_MATCH);
    }
}
```

- [ ] **Step 5: Write WorkerCredentialFilter tests**

Create `acl-worker/src/test/java/io/casehub/platform/acl/worker/WorkerCredentialFilterTest.java`:

```java
package io.casehub.platform.acl.worker;

import static org.mockito.Mockito.*;

import io.casehub.platform.api.acl.AclAction;
import io.casehub.platform.api.acl.ResourceId;
import io.casehub.platform.api.acl.WorkerAction;
import io.casehub.platform.api.acl.WorkerCredential;
import io.casehub.platform.api.acl.WorkerCredentialStore;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.core.UriInfo;
import java.time.Instant;
import java.util.Optional;
import java.util.Set;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

class WorkerCredentialFilterTest {

    private WorkerCredentialStore credentialStore;
    private WorkerScopeExtractor scopeExtractor;
    private CurrentPrincipal currentPrincipal;
    private WorkerCredentialFilter filter;
    private ContainerRequestContext ctx;

    @BeforeEach
    void setUp() {
        credentialStore = mock(WorkerCredentialStore.class);
        scopeExtractor = mock(WorkerScopeExtractor.class);
        currentPrincipal = mock(CurrentPrincipal.class);
        filter = new WorkerCredentialFilter(credentialStore, scopeExtractor, currentPrincipal);
        ctx = mock(ContainerRequestContext.class);
    }

    @Test
    void noHeader_passesThrough() {
        when(ctx.getHeaderString("X-Worker-Credential")).thenReturn(null);
        filter.filter(ctx);
        verify(ctx, never()).abortWith(any());
    }

    @Test
    void unknownToken_returns401() {
        when(ctx.getHeaderString("X-Worker-Credential")).thenReturn("bad-token");
        when(credentialStore.lookup("bad-token")).thenReturn(Optional.empty());
        filter.filter(ctx);
        verify(ctx).abortWith(argThat(r -> r.getStatus() == 401));
    }

    @Test
    void expiredToken_returns401() {
        when(ctx.getHeaderString("X-Worker-Credential")).thenReturn("tok");
        when(credentialStore.lookup("tok")).thenReturn(Optional.of(
            credential("tok", "actor", new ResourceId("case", "c1"), "tenant-1",
                Instant.now().minusSeconds(60))));
        filter.filter(ctx);
        verify(ctx).abortWith(argThat(r -> r.getStatus() == 401));
    }

    @Test
    void tenancyMismatch_returns403() {
        when(ctx.getHeaderString("X-Worker-Credential")).thenReturn("tok");
        when(credentialStore.lookup("tok")).thenReturn(Optional.of(
            credential("tok", "actor", new ResourceId("case", "c1"), "tenant-1",
                Instant.now().plusSeconds(3600))));
        when(currentPrincipal.tenancyId()).thenReturn("tenant-2");
        filter.filter(ctx);
        verify(ctx).abortWith(argThat(r -> r.getStatus() == 403));
    }

    @Test
    void scopeMismatch_returns403() {
        when(ctx.getHeaderString("X-Worker-Credential")).thenReturn("tok");
        when(credentialStore.lookup("tok")).thenReturn(Optional.of(
            credential("tok", "actor", new ResourceId("case", "c1"), "tenant-1",
                Instant.now().plusSeconds(3600))));
        when(currentPrincipal.tenancyId()).thenReturn("tenant-1");
        when(scopeExtractor.extractResourceId(ctx))
            .thenReturn(Optional.of(new ResourceId("case", "c2")));
        filter.filter(ctx);
        verify(ctx).abortWith(argThat(r -> r.getStatus() == 403));
    }

    @Test
    void scopeExtractorReturnsEmpty_scopeCheckSkipped() {
        var cred = credential("tok", "actor", new ResourceId("case", "c1"), "tenant-1",
            Instant.now().plusSeconds(3600));
        when(ctx.getHeaderString("X-Worker-Credential")).thenReturn("tok");
        when(credentialStore.lookup("tok")).thenReturn(Optional.of(cred));
        when(currentPrincipal.tenancyId()).thenReturn("tenant-1");
        when(scopeExtractor.extractResourceId(ctx)).thenReturn(Optional.empty());
        filter.filter(ctx);
        verify(ctx, never()).abortWith(any());
        verify(ctx).setProperty("workerCredential", cred);
    }

    @Test
    void validCredential_setsProperty() {
        var rid = new ResourceId("case", "c1");
        var cred = credential("tok", "actor", rid, "tenant-1", Instant.now().plusSeconds(3600));
        when(ctx.getHeaderString("X-Worker-Credential")).thenReturn("tok");
        when(credentialStore.lookup("tok")).thenReturn(Optional.of(cred));
        when(currentPrincipal.tenancyId()).thenReturn("tenant-1");
        when(scopeExtractor.extractResourceId(ctx)).thenReturn(Optional.of(rid));
        filter.filter(ctx);
        verify(ctx, never()).abortWith(any());
        verify(ctx).setProperty("workerCredential", cred);
    }

    private WorkerCredential credential(String token, String actorId,
            ResourceId resourceId, String tenancyId, Instant expiresAt) {
        return new WorkerCredential(token, actorId, resourceId, tenancyId,
            Set.of(new WorkerAction("READ_CONTEXT", AclAction.READ)),
            expiresAt, Instant.now());
    }
}
```

- [ ] **Step 6: Run tests — verify they fail**

Run: `mvn --batch-mode -pl acl-worker test -Dtest=WorkerCredentialFilterTest`
Expected: FAIL — `WorkerCredentialFilter` does not exist

- [ ] **Step 7: Implement WorkerCredentialFilter**

Create `acl-worker/src/main/java/io/casehub/platform/acl/worker/WorkerCredentialFilter.java`:

```java
package io.casehub.platform.acl.worker;

import io.casehub.platform.api.acl.WorkerCredentialStore;
import io.casehub.platform.api.identity.CurrentPrincipal;
import jakarta.annotation.Priority;
import jakarta.inject.Inject;
import jakarta.ws.rs.Priorities;
import jakarta.ws.rs.container.ContainerRequestContext;
import jakarta.ws.rs.container.ContainerRequestFilter;
import jakarta.ws.rs.core.Response;
import jakarta.ws.rs.ext.Provider;
import org.jboss.logging.Logger;

@Provider
@Priority(Priorities.AUTHENTICATION - 10)
public class WorkerCredentialFilter implements ContainerRequestFilter {

    private static final Logger LOG = Logger.getLogger(WorkerCredentialFilter.class);
    private static final String HEADER = "X-Worker-Credential";

    private final WorkerCredentialStore credentialStore;
    private final WorkerScopeExtractor scopeExtractor;
    private final CurrentPrincipal currentPrincipal;

    @Inject
    public WorkerCredentialFilter(
            WorkerCredentialStore credentialStore,
            WorkerScopeExtractor scopeExtractor,
            CurrentPrincipal currentPrincipal) {
        this.credentialStore = credentialStore;
        this.scopeExtractor = scopeExtractor;
        this.currentPrincipal = currentPrincipal;
    }

    @Override
    public void filter(ContainerRequestContext ctx) {
        String token = ctx.getHeaderString(HEADER);
        if (token == null) {
            return;
        }

        var credential = credentialStore.lookup(token);
        if (credential.isEmpty()) {
            ctx.abortWith(Response.status(401).entity("Invalid worker credential").build());
            return;
        }

        var cred = credential.get();
        if (cred.isExpired()) {
            ctx.abortWith(Response.status(401).entity("Worker credential expired").build());
            return;
        }

        String requestTenancy = currentPrincipal.tenancyId();
        if (!cred.tenancyId().equals(requestTenancy)) {
            LOG.warnf("Worker credential tenancy violation: credential=%s request=%s",
                cred.tenancyId(), requestTenancy);
            ctx.abortWith(Response.status(403)
                .entity("Credential not scoped for this tenant").build());
            return;
        }

        var requestResource = scopeExtractor.extractResourceId(ctx);
        if (requestResource.isPresent()
                && !cred.resourceId().equals(requestResource.get())) {
            LOG.warnf("Worker credential scope violation: credential=%s request=%s",
                cred.resourceId(), requestResource.get());
            ctx.abortWith(Response.status(403)
                .entity("Credential not scoped for this resource").build());
            return;
        }

        ctx.setProperty("workerCredential", cred);
    }
}
```

- [ ] **Step 8: Run tests — verify they pass**

Run: `mvn --batch-mode -pl acl-worker test -Dtest=WorkerCredentialFilterTest`
Expected: PASS

- [ ] **Step 9: Run full build**

Run: `mvn --batch-mode install`
Expected: PASS — all modules compile and test green

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/114/platform add acl-worker/ pom.xml
git -C /Users/mdproctor/claude/casehub/slots/114/platform commit -m "feat(#221): add acl-worker module — reusable WorkerCredentialFilter + WorkerScopeExtractor SPI"
```
