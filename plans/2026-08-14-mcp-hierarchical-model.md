# MCP Hierarchical Model Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/platform#228 — Phase 2: MCP hierarchical model
**Issue group:** #228, #229 (GraphQL foundation — done), #892, #347, #190, #394 (per-domain resolvers — later)

**Goal:** Build the MCP hierarchical model infrastructure — two fixed tools (`casehub_model` + `casehub_action`) that derive an agent-navigable operation catalog from GraphQL resolver annotations, with optional domain-level state enrichment.

**Architecture:** A Jandex scanner at CDI startup discovers `@GraphQLApi` classes annotated with `@McpDomain`, reads `@Query`/`@Mutation`/`@Subscription`/`@Description` annotations, and builds an internal model registry. Two `@Tool` methods on `@McpServer("casehub")` expose the model and dispatch operations via CDI-proxy-mediated reflection. `ModelEnricher` beans contribute live domain state.

**Tech Stack:** Quarkus 3.32.2, quarkus-mcp-server 1.11.1, MicroProfile GraphQL annotations, Jackson, Jandex, Java 21

## Global Constraints

- platform-api must remain zero-dependency: no Quarkus, no JPA, no casehubio imports. Pure Java only.
- `@McpDomain` and `ModelEnricher` go in platform-api (`io.casehub.platform.api.mcp`).
- mcp/ module uses `@McpServer("casehub")` per boundary-rules.md named server convention.
- Only methods annotated with `@Query` or `@Mutation` are dispatchable — security invariant.
- Resolver beans obtained as CDI proxies via `CDI.current().select()` for interceptor correctness.
- Jackson `ObjectMapper` configured with `JavaTimeModule` for ISO-8601 alignment with SmallRye GraphQL.
- All commits reference `Refs casehubio/platform#228`.
- IntelliJ MCP mandatory for .java file operations.

---

### Task 1: Platform API — @McpDomain annotation + ModelEnricher interface

**Repo:** casehub-platform
**Files:**
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/McpDomain.java`
- Create: `platform-api/src/main/java/io/casehub/platform/api/mcp/ModelEnricher.java`
- Test: `platform-api/src/test/java/io/casehub/platform/api/mcp/McpDomainTest.java`
- Test: `platform-api/src/test/java/io/casehub/platform/api/mcp/ModelEnricherTest.java`

**Interfaces:**
- Produces: `@McpDomain(String value)` — `@Retention(RUNTIME) @Target(TYPE)` marker annotation. Placed on GraphQL resolver classes and ModelEnricher beans to identify their domain.
- Produces: `ModelEnricher` — interface with `default String summary() { return ""; }` and `default Map<String, Object> state() { return Map.of(); }`. No `domain()` method — domain identity comes from `@McpDomain` on the implementing class.

- [ ] **Step 1: Write McpDomain annotation**

```java
package io.casehub.platform.api.mcp;

import java.lang.annotation.Retention;
import java.lang.annotation.RetentionPolicy;
import java.lang.annotation.Target;

import static java.lang.annotation.ElementType.TYPE;

@Retention(RetentionPolicy.RUNTIME)
@Target(TYPE)
public @interface McpDomain {
    String value();
}
```

- [ ] **Step 2: Write McpDomain test**

```java
package io.casehub.platform.api.mcp;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class McpDomainTest {

    @McpDomain("engine")
    static class AnnotatedClass {}

    static class UnannotatedClass {}

    @Test
    void annotationRetainedAtRuntime() {
        McpDomain ann = AnnotatedClass.class.getAnnotation(McpDomain.class);
        assertThat(ann).isNotNull();
        assertThat(ann.value()).isEqualTo("engine");
    }

    @Test
    void absentWhenNotAnnotated() {
        assertThat(UnannotatedClass.class.getAnnotation(McpDomain.class)).isNull();
    }
}
```

- [ ] **Step 3: Run test**

Run: `mvn --batch-mode test -pl platform-api -Dtest=McpDomainTest`
Expected: PASS

- [ ] **Step 4: Write ModelEnricher interface**

```java
package io.casehub.platform.api.mcp;

import java.util.Map;

public interface ModelEnricher {
    default String summary() { return ""; }
    default Map<String, Object> state() { return Map.of(); }
}
```

- [ ] **Step 5: Write ModelEnricher test**

```java
package io.casehub.platform.api.mcp;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.Map;

class ModelEnricherTest {

    @McpDomain("test")
    static class TestEnricher implements ModelEnricher {
        @Override
        public String summary() { return "Test domain"; }
        @Override
        public Map<String, Object> state() { return Map.of("count", 42); }
    }

    @McpDomain("minimal")
    static class MinimalEnricher implements ModelEnricher {}

    @Test
    void enricherWithOverrides() {
        var enricher = new TestEnricher();
        assertThat(enricher.summary()).isEqualTo("Test domain");
        assertThat(enricher.state()).containsEntry("count", 42);

        McpDomain domain = TestEnricher.class.getAnnotation(McpDomain.class);
        assertThat(domain.value()).isEqualTo("test");
    }

    @Test
    void enricherDefaults() {
        var enricher = new MinimalEnricher();
        assertThat(enricher.summary()).isEmpty();
        assertThat(enricher.state()).isEmpty();
    }
}
```

- [ ] **Step 6: Run tests**

Run: `mvn --batch-mode test -pl platform-api -Dtest=McpDomainTest,ModelEnricherTest`
Expected: PASS

- [ ] **Step 7: Verify platform-api has no new dependencies**

Check: `platform-api/pom.xml` — no new `<dependency>` entries added. Both types are pure Java.

- [ ] **Step 8: Commit**

```bash
git add platform-api/src/main/java/io/casehub/platform/api/mcp/McpDomain.java \
       platform-api/src/main/java/io/casehub/platform/api/mcp/ModelEnricher.java \
       platform-api/src/test/java/io/casehub/platform/api/mcp/McpDomainTest.java \
       platform-api/src/test/java/io/casehub/platform/api/mcp/ModelEnricherTest.java
git commit -m "feat(#228): @McpDomain annotation + ModelEnricher interface in platform-api"
```

---

### Task 2: Module skeleton — mcp/ with internal data model

**Repo:** casehub-platform
**Files:**
- Create: `mcp/pom.xml`
- Modify: `pom.xml` (add `<module>mcp</module>`)
- Create: `mcp/src/main/java/io/casehub/platform/mcp/OperationDescriptor.java`
- Create: `mcp/src/main/java/io/casehub/platform/mcp/EventDescriptor.java`
- Create: `mcp/src/main/java/io/casehub/platform/mcp/ParameterDescriptor.java`
- Create: `mcp/src/main/java/io/casehub/platform/mcp/DomainModel.java`
- Test: `mcp/src/test/java/io/casehub/platform/mcp/DomainModelTest.java`

**Interfaces:**
- Consumes: `@McpDomain`, `ModelEnricher` (from Task 1)
- Produces: Internal records used by scanner, registry, and dispatcher in later tasks.

- [ ] **Step 1: Create mcp/pom.xml**

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

    <artifactId>casehub-platform-mcp</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform MCP</name>
    <description>MCP hierarchical model — casehub_model + casehub_action tools.
        Auto-discovers GraphQL resolvers annotated with @McpDomain at startup via Jandex.
        No quarkus:build goal (library module).</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-api</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkiverse.mcp</groupId>
            <artifactId>quarkus-mcp-server-sse</artifactId>
            <version>1.11.1</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-smallrye-graphql</artifactId>
            <scope>provided</scope>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.core</groupId>
            <artifactId>jackson-databind</artifactId>
        </dependency>
        <dependency>
            <groupId>com.fasterxml.jackson.datatype</groupId>
            <artifactId>jackson-datatype-jsr310</artifactId>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
            <scope>provided</scope>
        </dependency>

        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.junit.jupiter</groupId>
            <artifactId>junit-jupiter</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.assertj</groupId>
            <artifactId>assertj-core</artifactId>
            <scope>test</scope>
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
            <plugin>
                <groupId>io.quarkus</groupId>
                <artifactId>quarkus-maven-plugin</artifactId>
                <version>${quarkus.platform.version}</version>
                <extensions>true</extensions>
                <executions>
                    <execution>
                        <goals>
                            <goal>generate-code</goal>
                            <goal>generate-code-tests</goal>
                        </goals>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>
```

- [ ] **Step 2: Add module to parent pom.xml**

Add `<module>mcp</module>` to the `<modules>` list in `pom.xml` (after `graphql-client`).

- [ ] **Step 3: Create ParameterDescriptor record**

```java
package io.casehub.platform.mcp;

import java.util.Map;

public record ParameterDescriptor(
        String name,
        String typeName,
        boolean required,
        String description,
        Map<String, String> fields) {

    public ParameterDescriptor(String name, String typeName, boolean required) {
        this(name, typeName, required, "", Map.of());
    }
}
```

- [ ] **Step 4: Create OperationDescriptor record**

```java
package io.casehub.platform.mcp;

import java.lang.reflect.Method;
import java.util.List;

public record OperationDescriptor(
        String name,
        OperationType type,
        String summary,
        List<ParameterDescriptor> params,
        String returnTypeName,
        Method method,
        Class<?> resolverClass) {

    public enum OperationType { QUERY, MUTATION }
}
```

- [ ] **Step 5: Create EventDescriptor record**

```java
package io.casehub.platform.mcp;

import java.util.List;

public record EventDescriptor(
        String name,
        String summary,
        List<ParameterDescriptor> params,
        String channel) {}
```

- [ ] **Step 6: Create DomainModel record**

```java
package io.casehub.platform.mcp;

import java.util.List;
import java.util.Map;

public record DomainModel(
        String name,
        String summary,
        List<OperationDescriptor> operations,
        List<EventDescriptor> events,
        Map<String, Object> state) {

    public long queryCount() {
        return operations.stream()
                .filter(op -> op.type() == OperationDescriptor.OperationType.QUERY)
                .count();
    }

    public long mutationCount() {
        return operations.stream()
                .filter(op -> op.type() == OperationDescriptor.OperationType.MUTATION)
                .count();
    }
}
```

- [ ] **Step 7: Write DomainModel test**

```java
package io.casehub.platform.mcp;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

import java.util.List;
import java.util.Map;

class DomainModelTest {

    @Test
    void countsOperationsByType() throws Exception {
        var query = new OperationDescriptor("caseById",
                OperationDescriptor.OperationType.QUERY, "Get case",
                List.of(), "CaseInstance", null, null);
        var mutation = new OperationDescriptor("startCase",
                OperationDescriptor.OperationType.MUTATION, "Start case",
                List.of(), "CaseInstance", null, null);

        var model = new DomainModel("engine", "Engine",
                List.of(query, mutation), List.of(), Map.of());

        assertThat(model.queryCount()).isEqualTo(1);
        assertThat(model.mutationCount()).isEqualTo(1);
        assertThat(model.operations()).hasSize(2);
    }

    @Test
    void emptyModel() {
        var model = new DomainModel("empty", "", List.of(), List.of(), Map.of());
        assertThat(model.queryCount()).isZero();
        assertThat(model.mutationCount()).isZero();
        assertThat(model.events()).isEmpty();
        assertThat(model.state()).isEmpty();
    }

    @Test
    void parameterDescriptorDefaults() {
        var param = new ParameterDescriptor("id", "UUID", true);
        assertThat(param.description()).isEmpty();
        assertThat(param.fields()).isEmpty();
    }
}
```

- [ ] **Step 8: Compile and test**

Run: `mvn --batch-mode test -pl mcp`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add mcp/pom.xml pom.xml \
       mcp/src/main/java/io/casehub/platform/mcp/ParameterDescriptor.java \
       mcp/src/main/java/io/casehub/platform/mcp/OperationDescriptor.java \
       mcp/src/main/java/io/casehub/platform/mcp/EventDescriptor.java \
       mcp/src/main/java/io/casehub/platform/mcp/DomainModel.java \
       mcp/src/test/java/io/casehub/platform/mcp/DomainModelTest.java
git commit -m "feat(#228): mcp/ module skeleton + internal data model records"
```

---

### Task 3: GraphQLModelScanner + ModelRegistry

**Repo:** casehub-platform
**Files:**
- Create: `mcp/src/main/java/io/casehub/platform/mcp/ModelRegistry.java`
- Create: `mcp/src/main/java/io/casehub/platform/mcp/GraphQLModelScanner.java`
- Create: `mcp/src/test/java/io/casehub/platform/mcp/TestQueryResolver.java`
- Create: `mcp/src/test/java/io/casehub/platform/mcp/TestMutationResolver.java`
- Create: `mcp/src/test/java/io/casehub/platform/mcp/TestModelEnricher.java`
- Test: `mcp/src/test/java/io/casehub/platform/mcp/GraphQLModelScannerTest.java`

**Interfaces:**
- Consumes: `@McpDomain` (Task 1), `ModelEnricher` (Task 1), internal records (Task 2)
- Produces: `ModelRegistry` — `getDomains() → List<DomainModel>`, `getDomain(String name) → Optional<DomainModel>`, `getOperation(String domain, String operation) → Optional<OperationDescriptor>`
- Produces: `GraphQLModelScanner` — `@Startup @ApplicationScoped` bean that populates ModelRegistry

- [ ] **Step 1: Write test resolver classes**

```java
package io.casehub.platform.mcp;

import io.casehub.platform.api.mcp.McpDomain;
import org.eclipse.microprofile.graphql.Description;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Mutation;
import org.eclipse.microprofile.graphql.Name;
import org.eclipse.microprofile.graphql.Query;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.UUID;

@McpDomain("test")
@GraphQLApi
@ApplicationScoped
public class TestQueryResolver {

    @Query
    @Description("Echo the input back")
    public String echo(@Name("message") @Description("The message to echo") String message) {
        return message;
    }

    @Query
    @Description("Return a greeting")
    public String hello() {
        return "Hello from CaseHub";
    }

    public String notExposed() {
        return "should not be invocable";
    }
}
```

```java
package io.casehub.platform.mcp;

import io.casehub.platform.api.mcp.McpDomain;
import org.eclipse.microprofile.graphql.Description;
import org.eclipse.microprofile.graphql.GraphQLApi;
import org.eclipse.microprofile.graphql.Mutation;

import jakarta.enterprise.context.ApplicationScoped;

@McpDomain("test")
@GraphQLApi
@ApplicationScoped
public class TestMutationResolver {

    @Mutation
    @Description("Store a value")
    public String store(String key, String value) {
        return key + "=" + value;
    }
}
```

```java
package io.casehub.platform.mcp;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.ModelEnricher;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.Map;

@McpDomain("test")
@ApplicationScoped
public class TestModelEnricher implements ModelEnricher {

    @Override
    public String summary() { return "Test domain for verification"; }

    @Override
    public Map<String, Object> state() { return Map.of("itemCount", 3); }
}
```

- [ ] **Step 2: Write ModelRegistry**

```java
package io.casehub.platform.mcp;

import jakarta.enterprise.context.ApplicationScoped;
import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class ModelRegistry {

    private final Map<String, DomainModel> domains = new ConcurrentHashMap<>();

    public void register(DomainModel model) {
        domains.put(model.name(), model);
    }

    public List<DomainModel> getDomains() {
        return List.copyOf(domains.values());
    }

    public Optional<DomainModel> getDomain(String name) {
        return Optional.ofNullable(domains.get(name));
    }

    public Optional<OperationDescriptor> getOperation(String domain, String operation) {
        return getDomain(domain)
                .flatMap(d -> d.operations().stream()
                        .filter(op -> op.name().equals(operation))
                        .findFirst());
    }
}
```

- [ ] **Step 3: Write GraphQLModelScanner**

The scanner discovers `@McpDomain` + `@GraphQLApi` classes via Jandex at startup, reads `@Query`/`@Mutation`/`@Subscription` methods, builds descriptors, merges enricher state, and registers with ModelRegistry.

```java
package io.casehub.platform.mcp;

import io.casehub.platform.api.mcp.McpDomain;
import io.casehub.platform.api.mcp.ModelEnricher;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import io.quarkus.runtime.StartupEvent;
import org.jboss.jandex.AnnotationInstance;
import org.jboss.jandex.ClassInfo;
import org.jboss.jandex.DotName;
import org.jboss.jandex.IndexView;
import org.jboss.logging.Logger;

import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@ApplicationScoped
public class GraphQLModelScanner {

    private static final Logger LOG = Logger.getLogger(GraphQLModelScanner.class);

    private static final DotName MCP_DOMAIN = DotName.createSimple(McpDomain.class);
    private static final DotName GRAPHQL_API = DotName.createSimple("org.eclipse.microprofile.graphql.GraphQLApi");
    private static final DotName QUERY = DotName.createSimple("org.eclipse.microprofile.graphql.Query");
    private static final DotName MUTATION = DotName.createSimple("org.eclipse.microprofile.graphql.Mutation");
    private static final DotName SUBSCRIPTION = DotName.createSimple("org.eclipse.microprofile.graphql.Subscription");
    private static final DotName DESCRIPTION = DotName.createSimple("org.eclipse.microprofile.graphql.Description");

    @Inject ModelRegistry registry;
    @Inject Instance<ModelEnricher> enrichers;
    @Inject io.quarkus.arc.ArcContainer arc;

    void onStartup(@Observes StartupEvent event) {
        scan();
    }

    void scan() {
        IndexView index = arc.beanManager().createInstance()
                .select(IndexView.class).stream().findFirst().orElse(null);
        if (index == null) {
            LOG.warn("Jandex index not available — MCP model will be empty");
            return;
        }

        Map<String, List<OperationDescriptor>> domainOps = new HashMap<>();
        Map<String, List<EventDescriptor>> domainEvents = new HashMap<>();

        for (AnnotationInstance ann : index.getAnnotations(MCP_DOMAIN)) {
            ClassInfo classInfo = ann.target().asClass();
            if (classInfo.annotation(GRAPHQL_API) == null) {
                LOG.warnf("@McpDomain on %s without @GraphQLApi — skipping", classInfo.name());
                continue;
            }

            String domain = ann.value().asString();
            Class<?> resolverClass;
            try {
                resolverClass = Thread.currentThread().getContextClassLoader()
                        .loadClass(classInfo.name().toString());
            } catch (ClassNotFoundException e) {
                LOG.warnf("Cannot load resolver class %s — skipping", classInfo.name());
                continue;
            }

            domainOps.computeIfAbsent(domain, k -> new ArrayList<>());
            domainEvents.computeIfAbsent(domain, k -> new ArrayList<>());

            for (Method method : resolverClass.getDeclaredMethods()) {
                if (method.isAnnotationPresent(org.eclipse.microprofile.graphql.Query.class)) {
                    domainOps.get(domain).add(buildOperation(method, resolverClass,
                            OperationDescriptor.OperationType.QUERY));
                } else if (method.isAnnotationPresent(org.eclipse.microprofile.graphql.Mutation.class)) {
                    domainOps.get(domain).add(buildOperation(method, resolverClass,
                            OperationDescriptor.OperationType.MUTATION));
                } else if (method.isAnnotationPresent(org.eclipse.microprofile.graphql.Subscription.class)) {
                    domainEvents.get(domain).add(buildEvent(method, domain));
                }
            }
        }

        Map<String, ModelEnricher> enricherMap = new HashMap<>();
        for (ModelEnricher enricher : enrichers) {
            McpDomain domainAnn = enricher.getClass().getAnnotation(McpDomain.class);
            if (domainAnn == null) {
                // CDI proxy — check superclass
                Class<?> cls = enricher.getClass();
                while (cls != null && domainAnn == null) {
                    domainAnn = cls.getAnnotation(McpDomain.class);
                    cls = cls.getSuperclass();
                }
            }
            if (domainAnn != null) {
                enricherMap.put(domainAnn.value(), enricher);
            } else {
                LOG.warnf("ModelEnricher %s has no @McpDomain — ignoring", enricher.getClass().getName());
            }
        }

        for (String domain : domainOps.keySet()) {
            ModelEnricher enricher = enricherMap.get(domain);
            String summary = enricher != null ? enricher.summary() : "";
            Map<String, Object> state = enricher != null ? enricher.state() : Map.of();

            DomainModel model = new DomainModel(domain, summary,
                    List.copyOf(domainOps.get(domain)),
                    List.copyOf(domainEvents.getOrDefault(domain, List.of())),
                    state);
            registry.register(model);
            LOG.infof("MCP domain '%s': %d operations, %d events",
                    domain, model.operations().size(), model.events().size());
        }

        for (String enricherDomain : enricherMap.keySet()) {
            if (!domainOps.containsKey(enricherDomain)) {
                LOG.warnf("ModelEnricher for domain '%s' has no matching resolvers", enricherDomain);
            }
        }
    }

    private OperationDescriptor buildOperation(Method method, Class<?> resolverClass,
                                                OperationDescriptor.OperationType type) {
        String summary = "";
        org.eclipse.microprofile.graphql.Description desc =
                method.getAnnotation(org.eclipse.microprofile.graphql.Description.class);
        if (desc != null) summary = desc.value();

        List<ParameterDescriptor> params = buildParams(method);
        return new OperationDescriptor(method.getName(), type, summary, params,
                method.getReturnType().getSimpleName(), method, resolverClass);
    }

    private EventDescriptor buildEvent(Method method, String domain) {
        String summary = "";
        org.eclipse.microprofile.graphql.Description desc =
                method.getAnnotation(org.eclipse.microprofile.graphql.Description.class);
        if (desc != null) summary = desc.value();

        String channel = domain + "-" + toKebabCase(method.getName());
        List<ParameterDescriptor> params = buildParams(method);
        return new EventDescriptor(method.getName(), summary, params, channel);
    }

    private List<ParameterDescriptor> buildParams(Method method) {
        List<ParameterDescriptor> params = new ArrayList<>();
        for (Parameter param : method.getParameters()) {
            String name = param.getName();
            org.eclipse.microprofile.graphql.Name nameAnn =
                    param.getAnnotation(org.eclipse.microprofile.graphql.Name.class);
            if (nameAnn != null) name = nameAnn.value();

            String description = "";
            org.eclipse.microprofile.graphql.Description descAnn =
                    param.getAnnotation(org.eclipse.microprofile.graphql.Description.class);
            if (descAnn != null) description = descAnn.value();

            Map<String, String> fields = expandFields(param.getType());
            boolean required = !param.getType().equals(java.util.Optional.class);

            params.add(new ParameterDescriptor(name, mapTypeName(param.getType()),
                    required, description, fields));
        }
        return params;
    }

    Map<String, String> expandFields(Class<?> type) {
        if (type.isPrimitive() || type == String.class || type == java.util.UUID.class
                || type == java.time.Instant.class || type.isEnum()
                || java.util.Map.class.isAssignableFrom(type)) {
            return Map.of();
        }
        Map<String, String> fields = new HashMap<>();
        if (type.isRecord()) {
            for (var component : type.getRecordComponents()) {
                fields.put(component.getName(), mapTypeName(component.getType()));
            }
        } else {
            for (var field : type.getDeclaredFields()) {
                if (java.lang.reflect.Modifier.isStatic(field.getModifiers())) continue;
                fields.put(field.getName(), mapTypeName(field.getType()));
            }
        }
        return fields;
    }

    String mapTypeName(Class<?> type) {
        if (type == String.class) return "String";
        if (type == java.util.UUID.class) return "UUID";
        if (type == java.time.Instant.class) return "Instant";
        if (type == int.class || type == Integer.class) return "Integer";
        if (type == long.class || type == Long.class) return "Long";
        if (type == boolean.class || type == Boolean.class) return "Boolean";
        if (type == double.class || type == Double.class) return "Double";
        if (java.util.Map.class.isAssignableFrom(type)) return "JSON";
        if (java.util.List.class.isAssignableFrom(type)) return "List";
        return type.getSimpleName();
    }

    static String toKebabCase(String camelCase) {
        return camelCase.replaceAll("([a-z])([A-Z])", "$1-$2").toLowerCase();
    }
}
```

- [ ] **Step 4: Write scanner test**

```java
package io.casehub.platform.mcp;

import io.quarkus.test.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class GraphQLModelScannerTest {

    @Inject ModelRegistry registry;

    @Test
    void scansTestDomain() {
        var domain = registry.getDomain("test");
        assertThat(domain).isPresent();
        assertThat(domain.get().name()).isEqualTo("test");
    }

    @Test
    void discoversQueryOperations() {
        var domain = registry.getDomain("test").orElseThrow();
        assertThat(domain.queryCount()).isEqualTo(2); // echo, hello
    }

    @Test
    void discoversMutationOperations() {
        var domain = registry.getDomain("test").orElseThrow();
        assertThat(domain.mutationCount()).isEqualTo(1); // store
    }

    @Test
    void operationHasDescription() {
        var echo = registry.getOperation("test", "echo").orElseThrow();
        assertThat(echo.summary()).isEqualTo("Echo the input back");
    }

    @Test
    void operationHasParams() {
        var echo = registry.getOperation("test", "echo").orElseThrow();
        assertThat(echo.params()).hasSize(1);
        assertThat(echo.params().get(0).name()).isEqualTo("message");
        assertThat(echo.params().get(0).description()).isEqualTo("The message to echo");
    }

    @Test
    void nonAnnotatedMethodNotExposed() {
        var notExposed = registry.getOperation("test", "notExposed");
        assertThat(notExposed).isEmpty();
    }

    @Test
    void enricherMergesSummaryAndState() {
        var domain = registry.getDomain("test").orElseThrow();
        assertThat(domain.summary()).isEqualTo("Test domain for verification");
        assertThat(domain.state()).containsEntry("itemCount", 3);
    }

    @Test
    void kebabCaseConversion() {
        assertThat(GraphQLModelScanner.toKebabCase("caseLifecycle")).isEqualTo("case-lifecycle");
        assertThat(GraphQLModelScanner.toKebabCase("workItemInboxUpdates")).isEqualTo("work-item-inbox-updates");
        assertThat(GraphQLModelScanner.toKebabCase("hello")).isEqualTo("hello");
    }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn --batch-mode test -pl mcp`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git add mcp/src/main/java/io/casehub/platform/mcp/ModelRegistry.java \
       mcp/src/main/java/io/casehub/platform/mcp/GraphQLModelScanner.java \
       mcp/src/test/java/io/casehub/platform/mcp/TestQueryResolver.java \
       mcp/src/test/java/io/casehub/platform/mcp/TestMutationResolver.java \
       mcp/src/test/java/io/casehub/platform/mcp/TestModelEnricher.java \
       mcp/src/test/java/io/casehub/platform/mcp/GraphQLModelScannerTest.java
git commit -m "feat(#228): GraphQLModelScanner + ModelRegistry — annotation-driven model assembly"
```

---

### Task 4: ReflectiveOperationDispatcher

**Repo:** casehub-platform
**Files:**
- Create: `mcp/src/main/java/io/casehub/platform/mcp/ReflectiveOperationDispatcher.java`
- Test: `mcp/src/test/java/io/casehub/platform/mcp/ReflectiveOperationDispatcherTest.java`

**Interfaces:**
- Consumes: `OperationDescriptor` (Task 2), `ModelRegistry` (Task 3)
- Produces: `dispatch(String domain, String operation, Map<String, Object> params) → Object` — resolves CDI proxy, deserializes params, invokes method, returns result.

- [ ] **Step 1: Write dispatcher**

```java
package io.casehub.platform.mcp;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.spi.CDI;
import jakarta.inject.Inject;

import java.lang.reflect.Method;
import java.lang.reflect.Parameter;
import java.util.Map;

@ApplicationScoped
public class ReflectiveOperationDispatcher {

    @Inject ModelRegistry registry;

    private final ObjectMapper mapper;

    public ReflectiveOperationDispatcher() {
        this.mapper = new ObjectMapper();
        this.mapper.registerModule(new JavaTimeModule());
    }

    public Object dispatch(String domain, String operation, Map<String, Object> params)
            throws Exception {
        OperationDescriptor op = registry.getOperation(domain, operation)
                .orElseThrow(() -> new IllegalArgumentException(
                        "Unknown operation: " + domain + "." + operation));

        Object resolverBean = CDI.current().select(op.resolverClass()).get();

        Method method = op.method();
        Parameter[] methodParams = method.getParameters();
        Object[] args = new Object[methodParams.length];

        if (params == null) params = Map.of();

        for (int i = 0; i < methodParams.length; i++) {
            Parameter param = methodParams[i];
            String paramName = param.getName();
            org.eclipse.microprofile.graphql.Name nameAnn =
                    param.getAnnotation(org.eclipse.microprofile.graphql.Name.class);
            if (nameAnn != null) paramName = nameAnn.value();

            Object rawValue = params.get(paramName);
            if (rawValue != null) {
                args[i] = mapper.convertValue(rawValue, param.getType());
            } else if (param.getType().isPrimitive()) {
                throw new IllegalArgumentException(
                        "Required parameter '" + paramName + "' is missing");
            }
        }

        return method.invoke(resolverBean, args);
    }
}
```

- [ ] **Step 2: Write dispatcher test**

```java
package io.casehub.platform.mcp;

import io.quarkus.test.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class ReflectiveOperationDispatcherTest {

    @Inject ReflectiveOperationDispatcher dispatcher;

    @Test
    void dispatchesQueryOperation() throws Exception {
        Object result = dispatcher.dispatch("test", "echo",
                Map.of("message", "ping"));
        assertThat(result).isEqualTo("ping");
    }

    @Test
    void dispatchesNoArgQuery() throws Exception {
        Object result = dispatcher.dispatch("test", "hello", Map.of());
        assertThat(result).isEqualTo("Hello from CaseHub");
    }

    @Test
    void dispatchesMutationOperation() throws Exception {
        Object result = dispatcher.dispatch("test", "store",
                Map.of("key", "a", "value", "b"));
        assertThat(result).isEqualTo("a=b");
    }

    @Test
    void rejectsNonAnnotatedMethod() {
        assertThatThrownBy(() -> dispatcher.dispatch("test", "notExposed", Map.of()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Unknown operation");
    }

    @Test
    void rejectsUnknownDomain() {
        assertThatThrownBy(() -> dispatcher.dispatch("nonexistent", "echo", Map.of()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Unknown operation");
    }

    @Test
    void rejectsUnknownOperation() {
        assertThatThrownBy(() -> dispatcher.dispatch("test", "doesNotExist", Map.of()))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("Unknown operation");
    }

    @Test
    void handlesNullParams() throws Exception {
        Object result = dispatcher.dispatch("test", "hello", null);
        assertThat(result).isEqualTo("Hello from CaseHub");
    }
}
```

- [ ] **Step 3: Run tests**

Run: `mvn --batch-mode test -pl mcp`
Expected: PASS (all scanner + dispatcher tests)

- [ ] **Step 4: Commit**

```bash
git add mcp/src/main/java/io/casehub/platform/mcp/ReflectiveOperationDispatcher.java \
       mcp/src/test/java/io/casehub/platform/mcp/ReflectiveOperationDispatcherTest.java
git commit -m "feat(#228): ReflectiveOperationDispatcher — CDI-proxy-mediated annotation-filtered dispatch"
```

---

### Task 5: CaseHubMcpTools — casehub_model + casehub_action

**Repo:** casehub-platform
**Files:**
- Create: `mcp/src/main/java/io/casehub/platform/mcp/CaseHubMcpTools.java`
- Test: `mcp/src/test/java/io/casehub/platform/mcp/CaseHubMcpToolsTest.java`

**Interfaces:**
- Consumes: `ModelRegistry` (Task 3), `ReflectiveOperationDispatcher` (Task 4)
- Produces: `casehub_model(@ToolArg String domain)` — tier 0 (domain list) or tier 1 (domain detail)
- Produces: `casehub_action(@ToolArg String domain, @ToolArg String operation, @ToolArg String params)` — execute operation

- [ ] **Step 1: Write CaseHubMcpTools**

```java
package io.casehub.platform.mcp;

import com.fasterxml.jackson.core.JsonProcessingException;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.datatype.jsr310.JavaTimeModule;
import io.quarkiverse.mcp.server.McpServer;
import io.quarkiverse.mcp.server.Tool;
import io.quarkiverse.mcp.server.ToolArg;
import io.quarkiverse.mcp.server.WrapBusinessError;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

@McpServer("casehub")
@WrapBusinessError({IllegalArgumentException.class, IllegalStateException.class})
@ApplicationScoped
public class CaseHubMcpTools {

    @Inject ModelRegistry registry;
    @Inject ReflectiveOperationDispatcher dispatcher;

    private final ObjectMapper mapper;

    public CaseHubMcpTools() {
        this.mapper = new ObjectMapper();
        this.mapper.registerModule(new JavaTimeModule());
    }

    @Tool(description = "Navigate the CaseHub operation catalog. "
            + "Call without domain for a domain list. "
            + "Call with domain for operation details.")
    public String casehub_model(
            @ToolArg(description = "Domain name to drill into (omit for domain list)")
            String domain) throws JsonProcessingException {

        if (domain == null || domain.isBlank()) {
            return mapper.writeValueAsString(buildTier0());
        }
        return mapper.writeValueAsString(buildTier1(domain));
    }

    @Tool(description = "Execute a CaseHub operation. "
            + "Use casehub_model first to discover available operations.")
    public String casehub_action(
            @ToolArg(description = "Domain name") String domain,
            @ToolArg(description = "Operation name") String operation,
            @ToolArg(description = "Operation parameters as JSON object") String params)
            throws Exception {

        Map<String, Object> paramMap = Map.of();
        if (params != null && !params.isBlank()) {
            paramMap = mapper.readValue(params, new TypeReference<>() {});
        }

        Object result = dispatcher.dispatch(domain, operation, paramMap);
        return mapper.writeValueAsString(result);
    }

    private Map<String, Object> buildTier0() {
        List<Map<String, Object>> domainList = registry.getDomains().stream()
                .map(d -> {
                    Map<String, Object> entry = new LinkedHashMap<>();
                    entry.put("name", d.name());
                    if (!d.summary().isEmpty()) entry.put("summary", d.summary());
                    entry.put("operationCount", d.operations().size());
                    if (!d.events().isEmpty()) entry.put("eventCount", d.events().size());
                    if (!d.state().isEmpty()) entry.put("state", d.state());
                    return entry;
                })
                .collect(Collectors.toList());
        return Map.of("domains", domainList);
    }

    private Map<String, Object> buildTier1(String domainName) {
        DomainModel domain = registry.getDomain(domainName)
                .orElseThrow(() -> new IllegalArgumentException("Unknown domain: " + domainName));

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("domain", domain.name());
        if (!domain.summary().isEmpty()) result.put("summary", domain.summary());
        if (!domain.state().isEmpty()) result.put("state", domain.state());

        List<Map<String, Object>> queries = domain.operations().stream()
                .filter(op -> op.type() == OperationDescriptor.OperationType.QUERY)
                .map(this::operationToMap)
                .collect(Collectors.toList());
        if (!queries.isEmpty()) result.put("queries", queries);

        List<Map<String, Object>> mutations = domain.operations().stream()
                .filter(op -> op.type() == OperationDescriptor.OperationType.MUTATION)
                .map(this::operationToMap)
                .collect(Collectors.toList());
        if (!mutations.isEmpty()) result.put("mutations", mutations);

        List<Map<String, Object>> events = domain.events().stream()
                .map(this::eventToMap)
                .collect(Collectors.toList());
        if (!events.isEmpty()) result.put("events", events);

        return result;
    }

    private Map<String, Object> operationToMap(OperationDescriptor op) {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("name", op.name());
        if (!op.summary().isEmpty()) map.put("summary", op.summary());
        if (!op.params().isEmpty()) map.put("params", op.params().stream()
                .map(this::paramToMap).collect(Collectors.toList()));
        map.put("returns", op.returnTypeName());
        return map;
    }

    private Map<String, Object> eventToMap(EventDescriptor event) {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("name", event.name());
        if (!event.summary().isEmpty()) map.put("summary", event.summary());
        if (!event.params().isEmpty()) map.put("params", event.params().stream()
                .map(this::paramToMap).collect(Collectors.toList()));
        map.put("delivery", "qhorus");
        map.put("channel", event.channel());
        return map;
    }

    private Map<String, Object> paramToMap(ParameterDescriptor param) {
        Map<String, Object> map = new LinkedHashMap<>();
        map.put("name", param.name());
        map.put("type", param.typeName());
        if (param.required()) map.put("required", true);
        if (!param.description().isEmpty()) map.put("description", param.description());
        if (!param.fields().isEmpty()) map.put("fields", param.fields());
        return map;
    }
}
```

- [ ] **Step 2: Write integration test**

```java
package io.casehub.platform.mcp;

import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import io.quarkus.test.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

@QuarkusTest
class CaseHubMcpToolsTest {

    @Inject CaseHubMcpTools tools;
    private final ObjectMapper mapper = new ObjectMapper();

    @Test
    void tier0ReturnsDomainList() throws Exception {
        String json = tools.casehub_model(null);
        Map<String, Object> result = mapper.readValue(json, new TypeReference<>() {});

        assertThat(result).containsKey("domains");
        List<Map<String, Object>> domains = (List<Map<String, Object>>) result.get("domains");
        assertThat(domains).anyMatch(d -> "test".equals(d.get("name")));
    }

    @Test
    void tier0IncludesEnricherState() throws Exception {
        String json = tools.casehub_model(null);
        Map<String, Object> result = mapper.readValue(json, new TypeReference<>() {});
        List<Map<String, Object>> domains = (List<Map<String, Object>>) result.get("domains");
        Map<String, Object> testDomain = domains.stream()
                .filter(d -> "test".equals(d.get("name"))).findFirst().orElseThrow();

        assertThat(testDomain).containsEntry("summary", "Test domain for verification");
        assertThat(testDomain).containsKey("state");
    }

    @Test
    void tier1ReturnsOperationDetail() throws Exception {
        String json = tools.casehub_model("test");
        Map<String, Object> result = mapper.readValue(json, new TypeReference<>() {});

        assertThat(result).containsEntry("domain", "test");
        assertThat(result).containsKey("queries");
        assertThat(result).containsKey("mutations");
    }

    @Test
    void tier1QueriesHaveParams() throws Exception {
        String json = tools.casehub_model("test");
        Map<String, Object> result = mapper.readValue(json, new TypeReference<>() {});
        List<Map<String, Object>> queries = (List<Map<String, Object>>) result.get("queries");
        Map<String, Object> echo = queries.stream()
                .filter(q -> "echo".equals(q.get("name"))).findFirst().orElseThrow();

        assertThat(echo).containsEntry("summary", "Echo the input back");
        assertThat(echo).containsKey("params");
    }

    @Test
    void tier1UnknownDomainThrows() {
        assertThatThrownBy(() -> tools.casehub_model("nonexistent"))
                .isInstanceOf(IllegalArgumentException.class);
    }

    @Test
    void actionExecutesQuery() throws Exception {
        String result = tools.casehub_action("test", "echo",
                "{\"message\": \"hello\"}");
        assertThat(result).isEqualTo("\"hello\"");
    }

    @Test
    void actionExecutesMutation() throws Exception {
        String result = tools.casehub_action("test", "store",
                "{\"key\": \"x\", \"value\": \"y\"}");
        assertThat(result).isEqualTo("\"x=y\"");
    }

    @Test
    void actionWithNullParams() throws Exception {
        String result = tools.casehub_action("test", "hello", null);
        assertThat(result).isEqualTo("\"Hello from CaseHub\"");
    }

    @Test
    void actionRejectsUnknownOperation() {
        assertThatThrownBy(() -> tools.casehub_action("test", "notExposed", null))
                .isInstanceOf(IllegalArgumentException.class);
    }
}
```

- [ ] **Step 3: Run all tests**

Run: `mvn --batch-mode test -pl mcp`
Expected: PASS (all scanner + dispatcher + tools tests)

- [ ] **Step 4: Run full platform build**

Run: `mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 5: Update CLAUDE.md module table**

Add the mcp/ module entry to the `## Modules` table in CLAUDE.md.

- [ ] **Step 6: Commit**

```bash
git add mcp/src/main/java/io/casehub/platform/mcp/CaseHubMcpTools.java \
       mcp/src/test/java/io/casehub/platform/mcp/CaseHubMcpToolsTest.java \
       CLAUDE.md
git commit -m "feat(#228): CaseHubMcpTools — casehub_model + casehub_action on @McpServer(\"casehub\")"
```

---

## Execution Order

```
Task 1  @McpDomain + ModelEnricher in platform-api
  ↓
Task 2  mcp/ module skeleton + internal records
  ↓
Task 3  GraphQLModelScanner + ModelRegistry
  ↓
Task 4  ReflectiveOperationDispatcher
  ↓
Task 5  CaseHubMcpTools integration
```

All tasks are sequential — each depends on the previous.
