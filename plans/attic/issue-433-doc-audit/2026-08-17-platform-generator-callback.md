# Platform Code Generator + SPI Callback System — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/platform#230 — Callback registration API and registry
**Issue group:** #230, #231 (platform), #893 (engine), #348 (work)

**Goal:** Build a Quarkus build-time code generator that produces GraphQL resolvers, DTOs, and callback decorators from annotated SPI interfaces, plus the callback registration infrastructure (registry, invoker, lease management).

**Architecture:** Annotations (`@PlatformQuery`, `@PlatformMutation`, `@CallbackEligible`) on SPI interfaces in api/ modules. Two Quarkus deployment extensions: one generates GraphQL resolvers + DTOs, the other generates callback `@Decorator` classes. Callback infrastructure uses lease + graceful deregister with `WorkerCredential` identity propagation. Shared pagination utilities in the existing `graphql/` module.

**Tech Stack:** Quarkus 3.32+, SmallRye GraphQL (MicroProfile GraphQL), Quarkus deployment extensions (Jandex, source generation), Jackson, virtual threads (Java 21)

## Global Constraints

- `platform-api/` must remain zero-dependency — annotations are pure Java
- Every commit references an issue
- IntelliJ MCP mandatory for .java file operations
- Virtual threads: blocking interfaces only (ADR-0005)
- Module tier structure: api/ is pure Java, JPA lives in persistence-* modules
- `@DefaultBean` for no-op SPI fallbacks (protocol PP-20260514)
- Separate modules for classpath-presence activation (protocol PP-20260603)

---

## Batch 1: Annotation model + shared utilities

Foundation that all subsequent batches depend on. After this batch: annotations exist in platform-api, pagination utility exists in graphql/, the build passes.

### Task 1: Annotation model in platform-api

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/PlatformQuery.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/PlatformMutation.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/CallbackEligible.java`
- Test: `platform-api/src/test/java/io/casehub/platform/api/mcp/PlatformAnnotationsTest.java`

**Interfaces:**
- Produces: `@PlatformQuery(String value)` — `@Target(METHOD)`, `@Retention(RUNTIME)`
- Produces: `@PlatformMutation(String value)` — `@Target(METHOD)`, `@Retention(RUNTIME)`
- Produces: `@CallbackEligible(String name, boolean fanOut)` — `@Target(TYPE)`, `@Retention(RUNTIME)`

- [ ] **Step 1: Write the test**

```java
package io.casehub.platform.api.mcp;

import org.junit.jupiter.api.Test;
import java.lang.annotation.ElementType;
import java.lang.annotation.RetentionPolicy;
import static org.assertj.core.api.Assertions.assertThat;

class PlatformAnnotationsTest {

    @PlatformQuery("List items")
    void queryMethod() {}

    @PlatformMutation("Create item")
    void mutationMethod() {}

    @CallbackEligible(name = "test-spi")
    interface TestSpi {}

    @CallbackEligible
    interface DefaultNameSpi {}

    @CallbackEligible(fanOut = false)
    interface SingleImplSpi {}

    @Test
    void platformQuery_hasRuntimeRetention() throws Exception {
        var ann = getClass().getDeclaredMethod("queryMethod")
                .getAnnotation(PlatformQuery.class);
        assertThat(ann).isNotNull();
        assertThat(ann.value()).isEqualTo("List items");
        assertThat(PlatformQuery.class.getAnnotation(
                java.lang.annotation.Retention.class).value())
                .isEqualTo(RetentionPolicy.RUNTIME);
        assertThat(PlatformQuery.class.getAnnotation(
                java.lang.annotation.Target.class).value())
                .containsExactly(ElementType.METHOD);
    }

    @Test
    void platformMutation_hasRuntimeRetention() throws Exception {
        var ann = getClass().getDeclaredMethod("mutationMethod")
                .getAnnotation(PlatformMutation.class);
        assertThat(ann).isNotNull();
        assertThat(ann.value()).isEqualTo("Create item");
    }

    @Test
    void callbackEligible_explicitName() {
        var ann = TestSpi.class.getAnnotation(CallbackEligible.class);
        assertThat(ann).isNotNull();
        assertThat(ann.name()).isEqualTo("test-spi");
        assertThat(ann.fanOut()).isTrue();
    }

    @Test
    void callbackEligible_defaultNameIsEmpty() {
        var ann = DefaultNameSpi.class.getAnnotation(CallbackEligible.class);
        assertThat(ann.name()).isEmpty();
    }

    @Test
    void callbackEligible_singleImpl() {
        var ann = SingleImplSpi.class.getAnnotation(CallbackEligible.class);
        assertThat(ann.fanOut()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl platform-api -Dtest=PlatformAnnotationsTest`
Expected: FAIL — annotations don't exist yet

- [ ] **Step 3: Create the annotations**

`PlatformQuery.java`:
```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PlatformQuery {
    String value() default "";
}
```

`PlatformMutation.java`:
```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface PlatformMutation {
    String value() default "";
}
```

`CallbackEligible.java`:
```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.ElementType;
import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

@Target(ElementType.TYPE)
@Retention(RetentionPolicy.RUNTIME)
public @interface CallbackEligible {
    String name() default "";
    boolean fanOut() default true;
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl platform-api -Dtest=PlatformAnnotationsTest`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add platform-api/src/main/java/io/casehub/platform/api/mcp/PlatformQuery.java platform-api/src/main/java/io/casehub/platform/api/mcp/PlatformMutation.java platform-api/src/main/java/io/casehub/platform/api/mcp/CallbackEligible.java platform-api/src/test/java/io/casehub/platform/api/mcp/PlatformAnnotationsTest.java
git commit -m "feat(#230): @PlatformQuery, @PlatformMutation, @CallbackEligible annotations"
```

### Task 2: PaginationHelper in graphql/

**Files:**
- Create: `graphql/src/main/java/io/casehub/platform/graphql/PaginationHelper.java`
- Test: `graphql/src/test/java/io/casehub/platform/graphql/PaginationHelperTest.java`

**Interfaces:**
- Consumes: `PageInput` (existing — `graphql/src/.../PageInput.java`), `PageInfo` (existing — `graphql/src/.../PageInfo.java`)
- Produces: `PaginationHelper.paginate(List<S> all, PageInput page, Function<S, T> mapper)` → `PageResult<T>`
- Produces: `PageResult<T>` record — `(List<T> items, PageInfo pageInfo)`

- [ ] **Step 1: Write the test**

```java
package io.casehub.platform.graphql;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.function.Function;
import static org.assertj.core.api.Assertions.assertThat;

class PaginationHelperTest {

    private final List<String> items = List.of("a", "b", "c", "d", "e");

    @Test
    void paginate_firstPage() {
        var result = PaginationHelper.paginate(items,
                new PageInput(0, 2, null), Function.identity());
        assertThat(result.items()).containsExactly("a", "b");
        assertThat(result.pageInfo().hasNext()).isTrue();
        assertThat(result.pageInfo().hasPrevious()).isFalse();
        assertThat(result.pageInfo().totalCount()).isEqualTo(5);
    }

    @Test
    void paginate_middlePage() {
        var result = PaginationHelper.paginate(items,
                new PageInput(2, 2, null), Function.identity());
        assertThat(result.items()).containsExactly("c", "d");
        assertThat(result.pageInfo().hasNext()).isTrue();
        assertThat(result.pageInfo().hasPrevious()).isTrue();
    }

    @Test
    void paginate_lastPage() {
        var result = PaginationHelper.paginate(items,
                new PageInput(4, 2, null), Function.identity());
        assertThat(result.items()).containsExactly("e");
        assertThat(result.pageInfo().hasNext()).isFalse();
        assertThat(result.pageInfo().hasPrevious()).isTrue();
    }

    @Test
    void paginate_nullPage_usesDefaults() {
        var result = PaginationHelper.paginate(items, null, Function.identity());
        assertThat(result.items()).containsExactly("a", "b", "c", "d", "e");
        assertThat(result.pageInfo().totalCount()).isEqualTo(5);
    }

    @Test
    void paginate_emptyList() {
        var result = PaginationHelper.paginate(List.of(), null, Function.identity());
        assertThat(result.items()).isEmpty();
        assertThat(result.pageInfo().totalCount()).isEqualTo(0);
        assertThat(result.pageInfo().hasNext()).isFalse();
        assertThat(result.pageInfo().hasPrevious()).isFalse();
    }

    @Test
    void paginate_withMapper() {
        var result = PaginationHelper.paginate(items,
                new PageInput(0, 3, null), String::toUpperCase);
        assertThat(result.items()).containsExactly("A", "B", "C");
    }

    @Test
    void paginate_offsetBeyondEnd() {
        var result = PaginationHelper.paginate(items,
                new PageInput(10, 2, null), Function.identity());
        assertThat(result.items()).isEmpty();
        assertThat(result.pageInfo().hasPrevious()).isTrue();
        assertThat(result.pageInfo().hasNext()).isFalse();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn --batch-mode test -pl graphql -Dtest=PaginationHelperTest`
Expected: FAIL

- [ ] **Step 3: Implement PaginationHelper and PageResult**

`PageResult.java`:
```java
package io.casehub.platform.graphql;

import java.util.List;

public record PageResult<T>(List<T> items, PageInfo pageInfo) {}
```

`PaginationHelper.java`:
```java
package io.casehub.platform.graphql;

import java.util.List;
import java.util.function.Function;

public final class PaginationHelper {

    private PaginationHelper() {}

    public static <S, T> PageResult<T> paginate(
            List<S> all, PageInput page, Function<S, T> mapper) {
        int offset = page != null && page.offset() != null ? page.offset() : 0;
        int limit = page != null && page.limit() != null ? page.limit() : 20;
        int total = all.size();
        int end = Math.min(offset + limit, total);

        List<T> items = offset < total
                ? all.subList(offset, end).stream().map(mapper).toList()
                : List.of();

        return new PageResult<>(items, new PageInfo(
                end < total, offset > 0, total, null));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn --batch-mode test -pl graphql -Dtest=PaginationHelperTest`
Expected: PASS

- [ ] **Step 5: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 6: Commit**

```bash
git add graphql/src/main/java/io/casehub/platform/graphql/PaginationHelper.java graphql/src/main/java/io/casehub/platform/graphql/PageResult.java graphql/src/test/java/io/casehub/platform/graphql/PaginationHelperTest.java
git commit -m "feat(#230): PaginationHelper + PageResult — shared pagination utility"
```

---

## Batch 2: Callback registration infrastructure

After this batch: `CallbackRegistry` SPI exists, in-memory implementation works, `CallbackInvoker` dispatches HTTP calls with retry. No generator yet — this is the runtime infrastructure the generated decorators will depend on.

### Task 3: CallbackRegistry SPI + in-memory implementation

**Files:**
- Create: `callback-api/pom.xml`
- Create: `callback-api/src/main/java/io/casehub/platform/api/callback/CallbackRegistration.java`
- Create: `callback-api/src/main/java/io/casehub/platform/api/callback/CallbackRegistrationRequest.java`
- Create: `callback-api/src/main/java/io/casehub/platform/api/callback/CallbackRegistry.java`
- Create: `callback-api/src/main/java/io/casehub/platform/api/callback/CallbackRegistered.java`
- Create: `callback-api/src/main/java/io/casehub/platform/api/callback/CallbackDeregistered.java`
- Create: `callback-inmem/pom.xml`
- Create: `callback-inmem/src/main/java/io/casehub/platform/callback/inmem/InMemoryCallbackRegistry.java`
- Create: `callback-inmem/src/test/java/io/casehub/platform/callback/inmem/InMemoryCallbackRegistryTest.java`
- Modify: `platform/src/main/java/io/casehub/platform/callback/NoOpCallbackRegistry.java` (new)
- Modify: `pom.xml` — add `callback-api`, `callback-inmem` modules

**Interfaces:**
- Produces: `CallbackRegistration` record — `(id, spiName, callbackUrl, credentialRef, tenancyId, timeoutMs, metadata, registeredAt, expiresAt, lastHeartbeatAt)`
- Produces: `CallbackRegistrationRequest` record — `(spiName, callbackUrl, credentialRef, tenancyId, timeoutMs, ttlSeconds, metadata)`
- Produces: `CallbackRegistry` SPI — `register(request)`, `deregister(id)`, `heartbeat(id)`, `findBySpi(spiName, tenancyId)`, `findById(id)`
- Produces: `CallbackRegistered` CDI event — `(registration)`
- Produces: `CallbackDeregistered` CDI event — `(registrationId, spiName)`

- [ ] **Step 1: Create callback-api module POM**

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

    <artifactId>casehub-platform-callback-api</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform Callback API</name>
    <description>Pure Java callback registration SPI — CallbackRegistration model,
        CallbackRegistry SPI. Zero dependencies.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
            <version>${project.version}</version>
        </dependency>
    </dependencies>

    <build>
        <plugins>
            <plugin>
                <groupId>io.smallrye</groupId>
                <artifactId>jandex-maven-plugin</artifactId>
                <version>${jandex-maven-plugin.version}</version>
                <executions>
                    <execution>
                        <id>make-index</id>
                        <goals><goal>jandex</goal></goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Create the SPI types**

`CallbackRegistration.java`:
```java
package io.casehub.platform.api.callback;

import java.time.Instant;
import java.util.Map;

public record CallbackRegistration(
        String id,
        String spiName,
        String callbackUrl,
        String credentialRef,
        String tenancyId,
        int timeoutMs,
        Map<String, String> metadata,
        Instant registeredAt,
        Instant expiresAt,
        Instant lastHeartbeatAt) {}
```

`CallbackRegistrationRequest.java`:
```java
package io.casehub.platform.api.callback;

import java.util.Map;

public record CallbackRegistrationRequest(
        String spiName,
        String callbackUrl,
        String credentialRef,
        String tenancyId,
        int timeoutMs,
        int ttlSeconds,
        Map<String, String> metadata) {

    public CallbackRegistrationRequest {
        if (spiName == null || spiName.isBlank())
            throw new IllegalArgumentException("spiName required");
        if (callbackUrl == null || callbackUrl.isBlank())
            throw new IllegalArgumentException("callbackUrl required");
        if (tenancyId == null || tenancyId.isBlank())
            throw new IllegalArgumentException("tenancyId required");
        if (ttlSeconds <= 0) ttlSeconds = 300;
        if (timeoutMs <= 0) timeoutMs = 30000;
        if (metadata == null) metadata = Map.of();
    }
}
```

`CallbackRegistry.java`:
```java
package io.casehub.platform.api.callback;

import java.util.List;
import java.util.Optional;

public interface CallbackRegistry {
    CallbackRegistration register(CallbackRegistrationRequest request);
    void deregister(String registrationId);
    void heartbeat(String registrationId);
    List<CallbackRegistration> findBySpi(String spiName, String tenancyId);
    Optional<CallbackRegistration> findById(String registrationId);
}
```

`CallbackRegistered.java`:
```java
package io.casehub.platform.api.callback;

public record CallbackRegistered(CallbackRegistration registration) {}
```

`CallbackDeregistered.java`:
```java
package io.casehub.platform.api.callback;

public record CallbackDeregistered(String registrationId, String spiName) {}
```

- [ ] **Step 3: Write the InMemoryCallbackRegistry test**

```java
package io.casehub.platform.callback.inmem;

import io.casehub.platform.api.callback.CallbackRegistrationRequest;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class InMemoryCallbackRegistryTest {

    private InMemoryCallbackRegistry registry;

    @BeforeEach
    void setUp() {
        registry = new InMemoryCallbackRegistry();
    }

    @Test
    void register_assignsIdAndExpiresAt() {
        var req = new CallbackRegistrationRequest(
                "worker-provisioner", "http://app:8080/callbacks",
                "app-cred", "tenant-1", 30000, 300, Map.of());
        var reg = registry.register(req);
        assertThat(reg.id()).isNotNull();
        assertThat(reg.spiName()).isEqualTo("worker-provisioner");
        assertThat(reg.expiresAt()).isAfter(reg.registeredAt());
    }

    @Test
    void register_upsertBySpiUrlTenant() {
        var req = new CallbackRegistrationRequest(
                "worker-provisioner", "http://app:8080/callbacks",
                "app-cred", "tenant-1", 30000, 300, Map.of());
        var first = registry.register(req);
        var second = registry.register(req);
        assertThat(second.id()).isEqualTo(first.id());
        assertThat(registry.findBySpi("worker-provisioner", "tenant-1")).hasSize(1);
    }

    @Test
    void findBySpi_filtersExpired() throws Exception {
        var req = new CallbackRegistrationRequest(
                "spi-a", "http://app:8080/cb",
                null, "tenant-1", 30000, 1, Map.of());
        registry.register(req);
        Thread.sleep(1100);
        assertThat(registry.findBySpi("spi-a", "tenant-1")).isEmpty();
    }

    @Test
    void heartbeat_extendsLease() {
        var req = new CallbackRegistrationRequest(
                "spi-a", "http://app:8080/cb",
                null, "tenant-1", 30000, 300, Map.of());
        var reg = registry.register(req);
        var originalExpiry = reg.expiresAt();
        registry.heartbeat(reg.id());
        var updated = registry.findById(reg.id()).orElseThrow();
        assertThat(updated.expiresAt()).isAfterOrEqualTo(originalExpiry);
    }

    @Test
    void deregister_removes() {
        var req = new CallbackRegistrationRequest(
                "spi-a", "http://app:8080/cb",
                null, "tenant-1", 30000, 300, Map.of());
        var reg = registry.register(req);
        registry.deregister(reg.id());
        assertThat(registry.findById(reg.id())).isEmpty();
    }

    @Test
    void findBySpi_orderedByRegisteredAt() {
        registry.register(new CallbackRegistrationRequest(
                "spi-a", "http://app1:8080/cb",
                null, "tenant-1", 30000, 300, Map.of()));
        registry.register(new CallbackRegistrationRequest(
                "spi-a", "http://app2:8080/cb",
                null, "tenant-1", 30000, 300, Map.of()));
        var results = registry.findBySpi("spi-a", "tenant-1");
        assertThat(results).hasSize(2);
        assertThat(results.get(0).registeredAt())
                .isBeforeOrEqualTo(results.get(1).registeredAt());
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn --batch-mode test -pl callback-inmem -Dtest=InMemoryCallbackRegistryTest`
Expected: FAIL — module doesn't exist yet

- [ ] **Step 5: Create callback-inmem module + implementation**

Create `callback-inmem/pom.xml` with dependencies on `callback-api` and `quarkus-arc`.

`InMemoryCallbackRegistry.java`:
```java
package io.casehub.platform.callback.inmem;

import io.casehub.platform.api.callback.*;
import io.casehub.platform.api.util.UUIDv7;
import jakarta.alternative.Priority;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Alternative;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;
import java.util.stream.Collectors;

@Alternative
@Priority(100)
@ApplicationScoped
public class InMemoryCallbackRegistry implements CallbackRegistry {

    private final Map<String, CallbackRegistration> registrations = new ConcurrentHashMap<>();
    private final Map<String, String> upsertKeys = new ConcurrentHashMap<>();

    @Override
    public CallbackRegistration register(CallbackRegistrationRequest request) {
        String upsertKey = request.spiName() + "|" + request.callbackUrl() + "|" + request.tenancyId();
        String existingId = upsertKeys.get(upsertKey);

        Instant now = Instant.now();
        String id = existingId != null ? existingId : UUIDv7.generate().toString();

        var reg = new CallbackRegistration(
                id, request.spiName(), request.callbackUrl(),
                request.credentialRef(), request.tenancyId(),
                request.timeoutMs(), request.metadata(),
                now, now.plusSeconds(request.ttlSeconds()), now);

        registrations.put(id, reg);
        upsertKeys.put(upsertKey, id);
        return reg;
    }

    @Override
    public void deregister(String registrationId) {
        var removed = registrations.remove(registrationId);
        if (removed != null) {
            String key = removed.spiName() + "|" + removed.callbackUrl() + "|" + removed.tenancyId();
            upsertKeys.remove(key);
        }
    }

    @Override
    public void heartbeat(String registrationId) {
        registrations.computeIfPresent(registrationId, (id, reg) -> {
            long ttl = reg.expiresAt().getEpochSecond() - reg.registeredAt().getEpochSecond();
            Instant now = Instant.now();
            return new CallbackRegistration(
                    reg.id(), reg.spiName(), reg.callbackUrl(),
                    reg.credentialRef(), reg.tenancyId(),
                    reg.timeoutMs(), reg.metadata(),
                    reg.registeredAt(), now.plusSeconds(ttl), now);
        });
    }

    @Override
    public List<CallbackRegistration> findBySpi(String spiName, String tenancyId) {
        Instant now = Instant.now();
        return registrations.values().stream()
                .filter(r -> r.spiName().equals(spiName))
                .filter(r -> r.tenancyId().equals(tenancyId))
                .filter(r -> r.expiresAt().isAfter(now))
                .sorted(Comparator.comparing(CallbackRegistration::registeredAt))
                .collect(Collectors.toList());
    }

    @Override
    public Optional<CallbackRegistration> findById(String registrationId) {
        return Optional.ofNullable(registrations.get(registrationId));
    }
}
```

- [ ] **Step 6: Add NoOpCallbackRegistry @DefaultBean in platform/**

```java
package io.casehub.platform.callback;

import io.casehub.platform.api.callback.*;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.Optional;

@DefaultBean
@ApplicationScoped
public class NoOpCallbackRegistry implements CallbackRegistry {
    @Override public CallbackRegistration register(CallbackRegistrationRequest r) { return null; }
    @Override public void deregister(String id) {}
    @Override public void heartbeat(String id) {}
    @Override public List<CallbackRegistration> findBySpi(String s, String t) { return List.of(); }
    @Override public Optional<CallbackRegistration> findById(String id) { return Optional.empty(); }
}
```

- [ ] **Step 7: Add modules to parent POM and platform/ POM dependency**

Add `<module>callback-api</module>` and `<module>callback-inmem</module>` to root `pom.xml`.
Add `callback-api` dependency to `platform/pom.xml`.

- [ ] **Step 8: Run tests**

Run: `mvn --batch-mode test -pl callback-inmem -Dtest=InMemoryCallbackRegistryTest`
Expected: PASS

- [ ] **Step 9: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 10: Commit**

```bash
git add callback-api/ callback-inmem/ platform/src/main/java/io/casehub/platform/callback/ pom.xml platform/pom.xml
git commit -m "feat(#230): CallbackRegistry SPI + InMemoryCallbackRegistry + NoOp @DefaultBean"
```

### Task 4: CircuitBreakerPolicy in governance + CallbackInvoker

**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/governance/CircuitBreakerPolicy.java`
- Modify: `governance/src/main/java/io/casehub/platform/governance/DefaultPolicyEnforcer.java` — add circuit breaker support
- Create: `governance/src/test/java/io/casehub/platform/governance/CircuitBreakerPolicyTest.java`
- Create: `callback/pom.xml`
- Create: `callback/src/main/java/io/casehub/platform/callback/CallbackInvoker.java`
- Create: `callback/src/main/java/io/casehub/platform/callback/LeaseReaper.java`
- Create: `callback/src/test/java/io/casehub/platform/callback/CallbackInvokerTest.java`
- Modify: `pom.xml` — add `callback` module

**Interfaces:**
- Consumes: `CallbackRegistry.findBySpi()`, `PolicyEnforcer.execute()`, `CredentialResolver.resolve()`, `WorkerCredentialStore.store()`
- Produces: `CircuitBreakerPolicy` record — `(failureThreshold, recoveryWindowMs)`
- Produces: `CallbackInvoker.invoke(CallbackRegistration, String methodName, Object[] args, Class<T> returnType)` → `T`
- Produces: `LeaseReaper` — `@Scheduled` every 60s, purges expired registrations

- [ ] **Step 1: Write CircuitBreakerPolicy test**

```java
package io.casehub.platform.governance;

import io.casehub.platform.api.governance.CircuitBreakerPolicy;
import io.casehub.platform.api.governance.ExecutionPolicy;
import io.casehub.platform.api.governance.RetryPolicy;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class CircuitBreakerPolicyTest {

    @Test
    void circuitBreakerPolicy_defaults() {
        var cb = new CircuitBreakerPolicy();
        assertThat(cb.failureThreshold()).isEqualTo(5);
        assertThat(cb.recoveryWindowMs()).isEqualTo(30000);
    }

    @Test
    void circuitBreakerPolicy_customValues() {
        var cb = new CircuitBreakerPolicy(3, 10000);
        assertThat(cb.failureThreshold()).isEqualTo(3);
        assertThat(cb.recoveryWindowMs()).isEqualTo(10000);
    }

    @Test
    void executionPolicy_withCircuitBreaker() {
        var policy = new ExecutionPolicy(5000,
                new RetryPolicy(3, 1000),
                new CircuitBreakerPolicy(5, 30000));
        assertThat(policy.circuitBreaker()).isNotNull();
        assertThat(policy.circuitBreaker().failureThreshold()).isEqualTo(5);
    }

    @Test
    void executionPolicy_noCircuitBreaker_backwardCompat() {
        var policy = new ExecutionPolicy(5000, new RetryPolicy());
        assertThat(policy.circuitBreaker()).isNull();
    }
}
```

- [ ] **Step 2: Create CircuitBreakerPolicy + update ExecutionPolicy**

`CircuitBreakerPolicy.java` in `platform-api/src/main/java/io/casehub/platform/api/governance/`:
```java
package io.casehub.platform.api.governance;

public record CircuitBreakerPolicy(int failureThreshold, int recoveryWindowMs) {
    public CircuitBreakerPolicy() {
        this(5, 30000);
    }
}
```

Update `ExecutionPolicy.java` to add optional `CircuitBreakerPolicy` field (add a new constructor overload, keep existing constructors for backward compat).

- [ ] **Step 3: Run governance tests**

Run: `mvn --batch-mode test -pl platform-api,governance`
Expected: PASS (backward-compatible change)

- [ ] **Step 4: Write CallbackInvoker test (with WireMock)**

Test that CallbackInvoker makes HTTP POST, handles retry on failure, and returns deserialized response. Use WireMock for HTTP mocking.

- [ ] **Step 5: Create callback/ module with CallbackInvoker + LeaseReaper**

`CallbackInvoker.java` — HTTP POST to `registration.callbackUrl()/{methodName}`, Jackson serialization, retry via `PolicyEnforcer`, `WorkerCredential` minting.

`LeaseReaper.java` — `@Scheduled(every = "60s")` that calls `callbackRegistry.findBySpi()` and removes expired entries.

- [ ] **Step 6: Run tests**

Run: `mvn --batch-mode test -pl callback`
Expected: PASS

- [ ] **Step 7: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git add platform-api/src/main/java/io/casehub/platform/api/governance/CircuitBreakerPolicy.java callback/ governance/ pom.xml
git commit -m "feat(#230): CircuitBreakerPolicy in governance + CallbackInvoker + LeaseReaper"
```

---

## Batch 3: GraphQL code generator

After this batch: the Quarkus deployment extension generates GraphQL resolvers + DTOs from annotated SPI interfaces. Tested with sample SPIs.

### Task 5: GraphQL generator deployment extension

**Files:**
- Create: `graphql-generator-deployment/pom.xml`
- Create: `graphql-generator-deployment/src/main/java/io/casehub/platform/graphql/generator/GraphQLServiceProcessor.java`
- Create: `graphql-generator-deployment/src/main/java/io/casehub/platform/graphql/generator/ResolverGenerator.java`
- Create: `graphql-generator-deployment/src/main/java/io/casehub/platform/graphql/generator/DtoGenerator.java`
- Create: `graphql-generator-deployment/src/main/java/io/casehub/platform/graphql/generator/ModuleBoundaryValidator.java`
- Create: `graphql-generator-deployment/src/test/java/io/casehub/platform/graphql/generator/GraphQLGeneratorTest.java`
- Create: `graphql-generator-deployment/src/test/java/io/casehub/platform/graphql/generator/test/` (sample SPIs)
- Modify: `pom.xml` — add module

**Interfaces:**
- Consumes: `@PlatformQuery`, `@PlatformMutation`, `@McpDomain` annotations (from platform-api)
- Produces: Generated `@GraphQLApi @McpDomain` resolver classes with `@Query`/`@Mutation`/`@Description`
- Produces: Generated `@Type` DTO records with `from()` factory methods

This task is substantial — the core generator. It produces Java source files at build time from Jandex-scanned annotations. The test harness uses sample SPI interfaces annotated with `@PlatformQuery`/`@PlatformMutation`/`@McpDomain`, verifies generated source compiles and functions correctly.

- [ ] **Step 1: Create module POM** (Quarkus deployment extension structure)
- [ ] **Step 2: Create sample test SPI interfaces** with `@McpDomain("test")` + `@PlatformQuery`/`@PlatformMutation`
- [ ] **Step 3: Write GraphQLGeneratorTest** — `@QuarkusTest` that verifies generated resolver is discoverable and operations work
- [ ] **Step 4: Run test to verify it fails**
- [ ] **Step 5: Implement GraphQLServiceProcessor** — `@BuildStep` that scans Jandex, groups by domain, generates .java source
- [ ] **Step 6: Implement ResolverGenerator** — produces resolver source code string from operation descriptors
- [ ] **Step 7: Implement DtoGenerator** — produces DTO record source from return types
- [ ] **Step 8: Implement ModuleBoundaryValidator** — Jandex check that return types are from api/ modules
- [ ] **Step 9: Test hand-written resolver skip** — add a hand-written resolver for one test method, verify generator skips it
- [ ] **Step 10: Run tests**
- [ ] **Step 11: Run full build**
- [ ] **Step 12: Commit**

```bash
git add graphql-generator-deployment/ pom.xml
git commit -m "feat(#230): GraphQL code generator — Quarkus deployment extension"
```

---

## Batch 4: Callback generator

After this batch: the callback deployment extension generates `@Decorator` classes from `@CallbackEligible` SPIs. Tested with sample SPIs.

### Task 6: Callback decorator generator

**Files:**
- Create: `callback-generator-deployment/pom.xml`
- Create: `callback-generator-deployment/src/main/java/io/casehub/platform/callback/generator/CallbackProxyProcessor.java`
- Create: `callback-generator-deployment/src/main/java/io/casehub/platform/callback/generator/CallbackDecoratorGenerator.java`
- Create: `callback-generator-deployment/src/main/java/io/casehub/platform/callback/generator/SerializationValidator.java`
- Create: `callback-generator-deployment/src/test/java/io/casehub/platform/callback/generator/CallbackGeneratorTest.java`
- Create: `callback-generator-deployment/src/test/java/io/casehub/platform/callback/generator/test/` (sample SPIs)
- Modify: `pom.xml` — add module

**Interfaces:**
- Consumes: `@CallbackEligible` annotation, `CallbackRegistry`, `CallbackInvoker`
- Produces: Generated `@Decorator @Priority(APPLICATION + 100)` classes implementing `@CallbackEligible` SPIs
- Produces: `SerializationValidator` — build-time check that all param/return types are Jackson-safe and from api/ modules

- [ ] **Step 1: Create module POM**
- [ ] **Step 2: Create sample @CallbackEligible SPI** — one fanOut=true (void method), one fanOut=false (return-value method)
- [ ] **Step 3: Write CallbackGeneratorTest** — verify generated decorator routes to CallbackInvoker when registrations exist, delegates when not
- [ ] **Step 4: Run test to verify it fails**
- [ ] **Step 5: Implement CallbackProxyProcessor** — `@BuildStep` scanning @CallbackEligible, generating decorators
- [ ] **Step 6: Implement CallbackDecoratorGenerator** — produces decorator source with fan-out/single-impl control flow
- [ ] **Step 7: Implement SerializationValidator** — build-time Jackson-safety check
- [ ] **Step 8: Test fanOut=true and fanOut=false** — verify different control flow
- [ ] **Step 9: Run tests**
- [ ] **Step 10: Run full build**
- [ ] **Step 11: Commit**

```bash
git add callback-generator-deployment/ pom.xml
git commit -m "feat(#230): Callback decorator generator — Quarkus deployment extension"
```

---

## Batch 5: Integration + MCP scanner verification

After this batch: generated resolvers appear in the MCP model, full end-to-end callback flow works, existing tests still pass.

### Task 7: End-to-end integration test

**Files:**
- Create: `mcp/src/test/java/io/casehub/platform/mcp/GeneratedResolverMcpTest.java`
- Create: `callback/src/test/java/io/casehub/platform/callback/CallbackEndToEndTest.java`

**Interfaces:**
- Consumes: All previous tasks — annotations, generators, callback infrastructure
- Produces: Integration tests proving: (1) generated resolvers appear in MCP model, (2) callback registration → invocation → response round-trip works

- [ ] **Step 1: Write GeneratedResolverMcpTest** — `@QuarkusTest` with sample SPI + generated resolver, verify `casehub_model` returns the domain and `casehub_action` dispatches
- [ ] **Step 2: Run test to verify it fails** (generator needs to run during Quarkus build)
- [ ] **Step 3: Wire test dependencies** — ensure graphql-generator-deployment is on the test classpath
- [ ] **Step 4: Run test to verify it passes**
- [ ] **Step 5: Write CallbackEndToEndTest** — register callback via registry, invoke SPI, verify HTTP call to WireMock, verify response returned
- [ ] **Step 6: Run callback test**
- [ ] **Step 7: Run existing MCP test suite** — verify no regressions

Run: `mvn --batch-mode test -pl mcp`
Expected: All 14+ existing tests PASS

- [ ] **Step 8: Run full build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git add mcp/src/test/java/ callback/src/test/java/
git commit -m "test(#230): end-to-end integration — generated resolvers in MCP + callback round-trip"
```

---

## References

- [2026-08-17-platform-generator-callback-design.md] — design spec this plan implements
- [2026-08-14-mcp-hierarchical-model-design.md] — MCP spec (scanner, tools)
- [decisions.md] — D8-D16 decision log
- [platform-api/src/main/java/io/casehub/platform/api/mcp/McpDomain.java] — existing annotation pattern
- [graphql/src/main/java/io/casehub/platform/graphql/PageInput.java] — existing pagination types
- [governance/src/main/java/io/casehub/platform/governance/DefaultPolicyEnforcer.java] — existing retry infrastructure
- [PP-20260514-engine-spi-noops-defaultbean] — @DefaultBean protocol
- [PP-20260603-fb05ef] — classpath-presence activation protocol
- [PP-20260808-340f01] — CDI @Decorator protocol for cross-cutting concerns
- [casehubio/platform#230] — callback registration API
- [casehubio/platform#231] — client-side auto-registration (out of scope, next issue)
