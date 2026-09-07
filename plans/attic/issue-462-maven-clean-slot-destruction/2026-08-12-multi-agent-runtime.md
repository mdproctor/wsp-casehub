# Multi-Agent Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #186 — Epic: Multi-Agent Runtime Support
**Issue group:** #186 (casehubio/devtown)

**Goal:** Enable multi-provider agent dispatch with native OpenAI, Gemini, and CLI agent support alongside the existing Claude provider.

**Architecture:** A `RoutingAgentProvider` implements `AgentProvider` and dispatches to `AgentBackend` implementations discovered via CDI `Instance`. CLI providers delegate subprocess management to an `AgentRuntime` SPI. Each provider is its own Maven module for classpath isolation.

**Tech Stack:** Java 21, Quarkus 3.32.2, Mutiny, CDI (ArC), OpenAI Java SDK, Google GenAI SDK, claude-code-sdk

## Global Constraints

- `agent-api/` must remain zero-dependency (pure Java + Mutiny only)
- Every module requires Jandex index (`jandex-maven-plugin`)
- Library modules: `generate-code` + `generate-code-tests` only — no `quarkus:build` goal
- Config prefix per module: `casehub.platform.agent.<provider-key>`
- IT tests gated by `@EnabledIfEnvironmentVariable`
- All new types in `io.casehub.platform.agent` package (or sub-packages per module)

---

### Task 1: SPI Types in agent-api/

**Files:**
- Create: `agent-api/src/main/java/io/casehub/platform/agent/AgentBackend.java`
- Create: `agent-api/src/main/java/io/casehub/platform/agent/AgentRuntime.java`
- Create: `agent-api/src/main/java/io/casehub/platform/agent/AgentRuntimeConfig.java`
- Create: `agent-api/src/main/java/io/casehub/platform/agent/AgentProcess.java`
- Modify: `agent-api/src/main/java/io/casehub/platform/agent/AgentSessionConfig.java`
- Modify: `agent-api/src/main/java/io/casehub/platform/agent/AgentSessionInit.java`
- Test: `agent-api/src/test/java/io/casehub/platform/agent/AgentBackendTest.java`
- Test: `agent-api/src/test/java/io/casehub/platform/agent/AgentRuntimeConfigTest.java`
- Modify: `agent-api/src/test/java/io/casehub/platform/agent/AgentSessionConfigTest.java`
- Modify: `agent-api/src/test/java/io/casehub/platform/agent/AgentSessionInitTest.java`

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces: `AgentBackend` (key(), invoke(), openSession()), `AgentRuntime` (spawn()), `AgentProcess` (stdin(), stdout(), stderr(), exitCode(), destroy(), destroyForcibly()), `AgentRuntimeConfig` (command, args, env, workingDirectory), `AgentSessionConfig.model()`, `AgentSessionInit.model()`

- [ ] **Step 1: Write failing test for AgentBackend**

```java
package io.casehub.platform.agent;

import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class AgentBackendTest {

    @Test
    void backendHasKeyInvokeAndOpenSession() {
        AgentBackend backend = new AgentBackend() {
            @Override public String key() { return "test"; }
            @Override public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                return Multi.createFrom().empty();
            }
            @Override public AgentSession openSession(AgentSessionInit init) {
                return null;
            }
        };
        assertThat(backend.key()).isEqualTo("test");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -pl agent-api test -Dtest=AgentBackendTest -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: FAIL — `AgentBackend` does not exist

- [ ] **Step 3: Create AgentBackend interface**

Use `ide_create_file` to create `agent-api/src/main/java/io/casehub/platform/agent/AgentBackend.java`:

```java
package io.casehub.platform.agent;

import io.smallrye.mutiny.Multi;

public interface AgentBackend {

    String key();

    Multi<AgentEvent> invoke(AgentSessionConfig config);

    AgentSession openSession(AgentSessionInit init);
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -pl agent-api test -Dtest=AgentBackendTest -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: PASS

- [ ] **Step 5: Write failing test for AgentRuntimeConfig**

```java
package io.casehub.platform.agent;

import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class AgentRuntimeConfigTest {

    @Test
    void commandIsRequired() {
        assertThatNullPointerException()
            .isThrownBy(() -> new AgentRuntimeConfig(null, List.of(), Map.of(), null))
            .withMessage("command");
    }

    @Test
    void argsAndEnvDefaultToEmpty() {
        var config = new AgentRuntimeConfig("codex", null, null, null);
        assertThat(config.args()).isEmpty();
        assertThat(config.env()).isEmpty();
    }

    @Test
    void defensiveCopy() {
        var args = new java.util.ArrayList<>(List.of("-p", "hello"));
        var env = new java.util.HashMap<>(Map.of("KEY", "val"));
        var config = new AgentRuntimeConfig("codex", args, env, Path.of("/tmp"));
        args.add("mutated");
        env.put("EXTRA", "mutated");
        assertThat(config.args()).hasSize(2);
        assertThat(config.env()).hasSize(1);
    }
}
```

- [ ] **Step 6: Create AgentRuntimeConfig, AgentRuntime, AgentProcess**

Use `ide_create_file` for each:

`AgentRuntimeConfig.java`:
```java
package io.casehub.platform.agent;

import java.nio.file.Path;
import java.util.List;
import java.util.Map;
import java.util.Objects;

public record AgentRuntimeConfig(
        String command,
        List<String> args,
        Map<String, String> env,
        Path workingDirectory
) {
    public AgentRuntimeConfig {
        Objects.requireNonNull(command, "command");
        args = args != null ? List.copyOf(args) : List.of();
        env = env != null ? Map.copyOf(env) : Map.of();
    }
}
```

`AgentRuntime.java`:
```java
package io.casehub.platform.agent;

public interface AgentRuntime {
    AgentProcess spawn(AgentRuntimeConfig config) throws java.io.IOException;
}
```

`AgentProcess.java`:
```java
package io.casehub.platform.agent;

import java.io.InputStream;
import java.io.OutputStream;
import java.util.concurrent.CompletableFuture;

public interface AgentProcess extends AutoCloseable {
    OutputStream stdin();
    InputStream stdout();
    InputStream stderr();
    CompletableFuture<Integer> exitCode();
    void destroy();
    void destroyForcibly();
}
```

- [ ] **Step 7: Run AgentRuntimeConfig test**

Run: `mvn -pl agent-api test -Dtest=AgentRuntimeConfigTest -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: PASS

- [ ] **Step 8: Add model field to AgentSessionConfig**

Use `ide_replace_member` on `AgentSessionConfig` to replace the record declaration. The new record adds a `model` parameter (nullable) and new factory methods:

```java
public record AgentSessionConfig(
        String systemPrompt,
        String userPrompt,
        List<AgentMcpServer> mcpServers,
        Duration timeout,
        String correlationId,
        String model
) {
    public AgentSessionConfig {
        Objects.requireNonNull(systemPrompt, "systemPrompt");
        Objects.requireNonNull(userPrompt, "userPrompt");
        mcpServers = mcpServers != null ? List.copyOf(mcpServers) : List.of();
    }

    public static AgentSessionConfig of(String systemPrompt, String userPrompt) {
        return new AgentSessionConfig(systemPrompt, userPrompt, List.of(), null, null, null);
    }

    public static AgentSessionConfig of(String systemPrompt, String userPrompt,
                                        Duration timeout) {
        return new AgentSessionConfig(systemPrompt, userPrompt, List.of(), timeout, null, null);
    }

    public static AgentSessionConfig of(String systemPrompt, String userPrompt,
                                        String model) {
        return new AgentSessionConfig(systemPrompt, userPrompt, List.of(), null, null, model);
    }
}
```

- [ ] **Step 9: Add model field to AgentSessionInit**

Use `ide_replace_member` on `AgentSessionInit`:

```java
public record AgentSessionInit(
        String systemPrompt,
        List<AgentMcpServer> mcpServers,
        Duration timeout,
        String correlationId,
        String model
) {
    public AgentSessionInit {
        Objects.requireNonNull(systemPrompt, "systemPrompt");
        mcpServers = mcpServers != null ? List.copyOf(mcpServers) : List.of();
    }

    public static AgentSessionInit of(String systemPrompt) {
        return new AgentSessionInit(systemPrompt, List.of(), null, null, null);
    }

    public static AgentSessionInit of(String systemPrompt, String model) {
        return new AgentSessionInit(systemPrompt, List.of(), null, null, model);
    }
}
```

- [ ] **Step 10: Update existing tests for model field**

Add model-field tests to `AgentSessionConfigTest`:
```java
@Test
void modelDefaultsToNull() {
    var config = AgentSessionConfig.of("sys", "user");
    assertThat(config.model()).isNull();
}

@Test
void modelSetViaFactory() {
    var config = AgentSessionConfig.of("sys", "user", "claude");
    assertThat(config.model()).isEqualTo("claude");
}
```

Add to `AgentSessionInitTest`:
```java
@Test
void modelDefaultsToNull() {
    var init = AgentSessionInit.of("sys");
    assertThat(init.model()).isNull();
}

@Test
void modelSetViaFactory() {
    var init = AgentSessionInit.of("sys", "openai");
    assertThat(init.model()).isEqualTo("openai");
}
```

Fix any existing tests that construct `AgentSessionConfig` or `AgentSessionInit` directly (add `null` for the new model parameter).

- [ ] **Step 11: Run all agent-api tests**

Run: `mvn -pl agent-api test -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 12: Fix compilation in downstream modules**

The model field addition changes the record constructor signature. Run `ide_diagnostics` across all modules. Fix any compilation errors by adding `null` as the model argument to `AgentSessionConfig` and `AgentSessionInit` constructors in:
- `agent-claude/` (ClaudeAgentClient, ClaudeAgentSession, tests)
- `agent-langchain4j/` (ChatModelAgentProvider, AgentProviderChatModel, tests)
- `agent-gate/` (tests)

Run: `mvn --batch-mode compile -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 13: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/116/platform add agent-api/src
git -C /Users/mdproctor/claude/casehub/slots/116/platform commit -m "feat: add AgentBackend, AgentRuntime SPIs and model field to config records

Refs casehubio/devtown#186"
```

---

### Task 2: agent-runtime/ — SubprocessRuntime

**Files:**
- Create: `agent-runtime/pom.xml`
- Create: `agent-runtime/src/main/java/io/casehub/platform/agent/runtime/SubprocessRuntime.java`
- Create: `agent-runtime/src/main/java/io/casehub/platform/agent/runtime/LocalAgentProcess.java`
- Test: `agent-runtime/src/test/java/io/casehub/platform/agent/runtime/SubprocessRuntimeTest.java`
- Test: `agent-runtime/src/test/java/io/casehub/platform/agent/runtime/LocalAgentProcessTest.java`
- Modify: `pom.xml` (root — add module)

**Interfaces:**
- Consumes: `AgentRuntime`, `AgentProcess`, `AgentRuntimeConfig` from Task 1
- Produces: `SubprocessRuntime` @ApplicationScoped, `LocalAgentProcess` (package-private)

- [ ] **Step 1: Create module POM**

Write `agent-runtime/pom.xml`:

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

    <artifactId>casehub-platform-agent-runtime</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform Agent Runtime</name>
    <description>SubprocessRuntime — local process execution for CLI agent providers.
        No quarkus:build goal — library module.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <!-- Test -->
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

- [ ] **Step 2: Add module to root POM**

Add `<module>agent-runtime</module>` to root `pom.xml` after `<module>testing</module>`.

- [ ] **Step 3: Write failing test for SubprocessRuntime**

```java
package io.casehub.platform.agent.runtime;

import io.casehub.platform.agent.AgentProcess;
import io.casehub.platform.agent.AgentRuntimeConfig;
import org.junit.jupiter.api.Test;
import java.io.IOException;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class SubprocessRuntimeTest {

    @Test
    void spawnEchoProcess() throws IOException {
        var runtime = new SubprocessRuntime();
        var config = new AgentRuntimeConfig("echo", List.of("hello"), Map.of(), null);
        try (AgentProcess process = runtime.spawn(config)) {
            String output = new String(process.stdout().readAllBytes()).trim();
            assertThat(output).isEqualTo("hello");
            int exitCode = process.exitCode().join();
            assertThat(exitCode).isZero();
        }
    }

    @Test
    void destroyTerminatesProcess() throws IOException {
        var runtime = new SubprocessRuntime();
        var config = new AgentRuntimeConfig("sleep", List.of("60"), Map.of(), null);
        AgentProcess process = runtime.spawn(config);
        process.destroy();
        int exitCode = process.exitCode().join();
        assertThat(exitCode).isNotZero();
    }

    @Test
    void envPassedToProcess() throws IOException {
        var runtime = new SubprocessRuntime();
        var config = new AgentRuntimeConfig("sh", List.of("-c", "echo $TEST_VAR"),
                Map.of("TEST_VAR", "hello_runtime"), null);
        try (AgentProcess process = runtime.spawn(config)) {
            String output = new String(process.stdout().readAllBytes()).trim();
            assertThat(output).isEqualTo("hello_runtime");
        }
    }
}
```

- [ ] **Step 4: Implement SubprocessRuntime and LocalAgentProcess**

`SubprocessRuntime.java`:
```java
package io.casehub.platform.agent.runtime;

import io.casehub.platform.agent.AgentProcess;
import io.casehub.platform.agent.AgentRuntime;
import io.casehub.platform.agent.AgentRuntimeConfig;
import jakarta.enterprise.context.ApplicationScoped;
import java.io.IOException;
import java.util.ArrayList;
import java.util.List;

@ApplicationScoped
public class SubprocessRuntime implements AgentRuntime {

    @Override
    public AgentProcess spawn(AgentRuntimeConfig config) throws IOException {
        List<String> cmd = new ArrayList<>();
        cmd.add(config.command());
        cmd.addAll(config.args());
        ProcessBuilder pb = new ProcessBuilder(cmd);
        if (config.workingDirectory() != null) {
            pb.directory(config.workingDirectory().toFile());
        }
        pb.environment().putAll(config.env());
        return new LocalAgentProcess(pb.start());
    }
}
```

`LocalAgentProcess.java`:
```java
package io.casehub.platform.agent.runtime;

import io.casehub.platform.agent.AgentProcess;
import java.io.InputStream;
import java.io.OutputStream;
import java.util.concurrent.CompletableFuture;

class LocalAgentProcess implements AgentProcess {

    private final Process process;

    LocalAgentProcess(Process process) {
        this.process = process;
    }

    @Override public OutputStream stdin()  { return process.getOutputStream(); }
    @Override public InputStream stdout()  { return process.getInputStream(); }
    @Override public InputStream stderr()  { return process.getErrorStream(); }

    @Override
    public CompletableFuture<Integer> exitCode() {
        return process.onExit().thenApply(Process::exitValue);
    }

    @Override public void destroy()         { process.destroy(); }
    @Override public void destroyForcibly() { process.destroyForcibly(); }
    @Override public void close()           { destroy(); }
}
```

- [ ] **Step 5: Run tests**

Run: `mvn -pl agent-runtime test -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/116/platform add agent-runtime/ pom.xml
git -C /Users/mdproctor/claude/casehub/slots/116/platform commit -m "feat: add agent-runtime module with SubprocessRuntime

Refs casehubio/devtown#186"
```

---

### Task 3: agent-router/ — RoutingAgentProvider

**Files:**
- Create: `agent-router/pom.xml`
- Create: `agent-router/src/main/java/io/casehub/platform/agent/router/RoutingAgentProvider.java`
- Create: `agent-router/src/main/java/io/casehub/platform/agent/router/RoutingAgentProperties.java`
- Test: `agent-router/src/test/java/io/casehub/platform/agent/router/RoutingAgentProviderTest.java`
- Modify: `pom.xml` (root — add module)

**Interfaces:**
- Consumes: `AgentProvider`, `AgentBackend`, `AgentSessionConfig.model()`, `AgentSessionInit.model()` from Task 1
- Produces: `RoutingAgentProvider` @ApplicationScoped implements AgentProvider. Config: `casehub.platform.agent.default-backend` (default: "claude")

- [ ] **Step 1: Create module POM**

Write `agent-router/pom.xml` — same structure as agent-claude/pom.xml but with `casehub-platform-agent-api` dependency only. Add `quarkus-junit5` for test scope.

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

    <artifactId>casehub-platform-agent-router</artifactId>
    <packaging>jar</packaging>
    <name>CaseHub Platform Agent Router</name>
    <description>RoutingAgentProvider — dispatches to AgentBackend implementations by provider key.
        No quarkus:build goal — library module.</description>

    <dependencies>
        <dependency>
            <groupId>io.casehub</groupId>
            <artifactId>casehub-platform-agent-api</artifactId>
            <version>${project.version}</version>
        </dependency>
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-arc</artifactId>
        </dependency>
        <!-- Test -->
        <dependency>
            <groupId>io.quarkus</groupId>
            <artifactId>quarkus-junit5</artifactId>
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

- [ ] **Step 2: Add module to root POM**

Add `<module>agent-router</module>` after `<module>agent-langchain4j</module>` and before `<module>agent-gate</module>`.

- [ ] **Step 3: Write failing test for RoutingAgentProvider**

```java
package io.casehub.platform.agent.router;

import io.casehub.platform.agent.*;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class RoutingAgentProviderTest {

    static AgentBackend stubBackend(String key) {
        return new AgentBackend() {
            @Override public String key() { return key; }
            @Override public Multi<AgentEvent> invoke(AgentSessionConfig config) {
                return Multi.createFrom().item(new AgentEvent.TextDelta("from-" + key));
            }
            @Override public AgentSession openSession(AgentSessionInit init) { return null; }
        };
    }

    @Test
    void routesByModelKey() {
        var router = new RoutingAgentProvider(
                List.of(stubBackend("claude"), stubBackend("openai")), "claude");
        var config = AgentSessionConfig.of("sys", "user", "openai");
        var events = router.invoke(config).collect().asList().await().indefinitely();
        assertThat(events).hasSize(1);
        assertThat(((AgentEvent.TextDelta) events.get(0)).text()).isEqualTo("from-openai");
    }

    @Test
    void nullModelUsesDefault() {
        var router = new RoutingAgentProvider(
                List.of(stubBackend("claude"), stubBackend("openai")), "claude");
        var config = AgentSessionConfig.of("sys", "user");
        var events = router.invoke(config).collect().asList().await().indefinitely();
        assertThat(((AgentEvent.TextDelta) events.get(0)).text()).isEqualTo("from-claude");
    }

    @Test
    void unknownKeyWithCatchAllFallsThrough() {
        var router = new RoutingAgentProvider(
                List.of(stubBackend("claude"), stubBackend("langchain4j")), "claude");
        var config = AgentSessionConfig.of("sys", "user", "mistral");
        var events = router.invoke(config).collect().asList().await().indefinitely();
        assertThat(((AgentEvent.TextDelta) events.get(0)).text()).isEqualTo("from-langchain4j");
    }

    @Test
    void unknownKeyWithNoCatchAllThrows() {
        var router = new RoutingAgentProvider(
                List.of(stubBackend("claude")), "claude");
        var config = AgentSessionConfig.of("sys", "user", "mistral");
        assertThatThrownBy(() -> router.invoke(config))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("mistral");
    }

    @Test
    void noDefaultBackendThrowsOnNullModel() {
        var router = new RoutingAgentProvider(
                List.of(stubBackend("openai")), "claude");
        var config = AgentSessionConfig.of("sys", "user");
        assertThatThrownBy(() -> router.invoke(config))
                .isInstanceOf(IllegalStateException.class)
                .hasMessageContaining("No default backend");
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn -pl agent-router test -Dtest=RoutingAgentProviderTest -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: FAIL — classes don't exist

- [ ] **Step 5: Implement RoutingAgentProvider and RoutingAgentProperties**

`RoutingAgentProvider.java`:
```java
package io.casehub.platform.agent.router;

import io.casehub.platform.agent.*;
import io.smallrye.mutiny.Multi;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Any;
import jakarta.enterprise.inject.Instance;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.util.HashMap;
import java.util.Map;

@ApplicationScoped
public class RoutingAgentProvider implements AgentProvider {

    private static final Logger LOG = Logger.getLogger(RoutingAgentProvider.class);

    private final Map<String, AgentBackend> backends;
    private final AgentBackend defaultBackend;

    @Inject
    public RoutingAgentProvider(@Any Instance<AgentBackend> backends,
                                RoutingAgentProperties properties) {
        this.backends = new HashMap<>();
        AgentBackend fallback = null;
        for (AgentBackend backend : backends) {
            this.backends.put(backend.key(), backend);
            if (backend.key().equals(properties.defaultBackend())) {
                fallback = backend;
            }
        }
        this.defaultBackend = fallback;
        LOG.infof("Agent router initialized: %d backends [%s], default=%s",
                this.backends.size(), String.join(", ", this.backends.keySet()),
                properties.defaultBackend());
    }

    RoutingAgentProvider(Iterable<AgentBackend> backends, String defaultKey) {
        this.backends = new HashMap<>();
        AgentBackend fallback = null;
        for (AgentBackend backend : backends) {
            this.backends.put(backend.key(), backend);
            if (backend.key().equals(defaultKey)) {
                fallback = backend;
            }
        }
        this.defaultBackend = fallback;
    }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        return resolve(config.model()).invoke(config);
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        return resolve(init.model()).openSession(init);
    }

    private AgentBackend resolve(String model) {
        if (model == null) {
            if (defaultBackend == null) {
                throw new IllegalStateException(
                        "No default backend configured — set casehub.platform.agent.default-backend");
            }
            return defaultBackend;
        }
        AgentBackend backend = backends.get(model);
        if (backend != null) return backend;
        AgentBackend catchAll = backends.get("langchain4j");
        if (catchAll != null) return catchAll;
        throw new IllegalArgumentException("No backend for key: " + model +
                ". Available: " + backends.keySet());
    }
}
```

`RoutingAgentProperties.java`:
```java
package io.casehub.platform.agent.router;

import io.smallrye.config.ConfigMapping;
import io.smallrye.config.WithDefault;

@ConfigMapping(prefix = "casehub.platform.agent")
public interface RoutingAgentProperties {
    @WithDefault("claude")
    String defaultBackend();
}
```

- [ ] **Step 6: Run tests**

Run: `mvn -pl agent-router test -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/116/platform add agent-router/ pom.xml
git -C /Users/mdproctor/claude/casehub/slots/116/platform commit -m "feat: add agent-router module with RoutingAgentProvider

Refs casehubio/devtown#186"
```

---

### Task 4: Refactor agent-claude/ to AgentBackend

**Files:**
- Modify: `agent-claude/src/main/java/io/casehub/platform/agent/claude/ClaudeAgentProvider.java`
- Modify: `agent-claude/pom.xml`
- Modify: `agent-claude/src/test/java/io/casehub/platform/agent/claude/ClaudeAgentClientTest.java`
- Modify: `agent-claude/src/test/java/io/casehub/platform/agent/claude/ClaudeAgentClientOpenSessionTest.java`

**Interfaces:**
- Consumes: `AgentBackend` from Task 1
- Produces: `ClaudeAgentProvider` implements `AgentBackend` with key `"claude"`

- [ ] **Step 1: Write test asserting key() returns "claude"**

Add to an existing test class or create a new one:
```java
@Test
void keyIsClaude() {
    assertThat(provider.key()).isEqualTo("claude");
}
```

Where `provider` is an instance of `ClaudeAgentProvider`.

- [ ] **Step 2: Change implements clause and add key()**

Use `ide_edit_member` on `ClaudeAgentProvider`:
- Change `implements AgentProvider` → `implements AgentBackend`
- Remove `@Alternative` and `@Priority(10)` annotations
- Add import for `AgentBackend`
- Add `@Override public String key() { return "claude"; }`

- [ ] **Step 3: Add agent-router dependency to POM**

Add to `agent-claude/pom.xml`:
```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-platform-agent-router</artifactId>
    <version>${project.version}</version>
</dependency>
```

- [ ] **Step 4: Run tests**

Run: `mvn -pl agent-claude test -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/116/platform add agent-claude/
git -C /Users/mdproctor/claude/casehub/slots/116/platform commit -m "refactor: agent-claude implements AgentBackend

ClaudeAgentProvider changes from AgentProvider to AgentBackend.
Removes @Alternative @Priority(10) — discovered by router via Instance<AgentBackend>.

Refs casehubio/devtown#186"
```

---

### Task 5: Refactor agent-langchain4j/ to AgentBackend

**Files:**
- Modify: `agent-langchain4j/src/main/java/io/casehub/platform/agent/langchain4j/ChatModelAgentProvider.java`
- Modify: `agent-langchain4j/pom.xml`
- Modify: tests as needed

**Interfaces:**
- Consumes: `AgentBackend` from Task 1
- Produces: `ChatModelAgentProvider` implements `AgentBackend` with key `"langchain4j"`

- [ ] **Step 1: Write test asserting key() returns "langchain4j"**

- [ ] **Step 2: Change implements clause and add key()**

Use `ide_edit_member` on `ChatModelAgentProvider`:
- Change `implements AgentProvider` → `implements AgentBackend`
- Remove `@Alternative` and `@Priority(1)` annotations
- Add `@Override public String key() { return "langchain4j"; }`

- [ ] **Step 3: Add agent-router dependency to POM**

- [ ] **Step 4: Run tests**

Run: `mvn -pl agent-langchain4j test -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/116/platform add agent-langchain4j/
git -C /Users/mdproctor/claude/casehub/slots/116/platform commit -m "refactor: agent-langchain4j implements AgentBackend

ChatModelAgentProvider is now the catch-all fallback with key 'langchain4j'.
Removes @Alternative @Priority(1).

Refs casehubio/devtown#186"
```

---

### Task 6: agent-openai/ — OpenAI API Provider

**Files:**
- Create: `agent-openai/pom.xml`
- Create: `agent-openai/src/main/java/io/casehub/platform/agent/openai/OpenAiAgentBackend.java`
- Create: `agent-openai/src/main/java/io/casehub/platform/agent/openai/OpenAiAgentProperties.java`
- Create: `agent-openai/src/main/java/io/casehub/platform/agent/openai/OpenAiAgentSession.java`
- Test: `agent-openai/src/test/java/io/casehub/platform/agent/openai/OpenAiAgentBackendTest.java`
- Modify: `pom.xml` (root — add module)

**Interfaces:**
- Consumes: `AgentBackend`, `AgentSessionConfig`, `AgentEvent` from Task 1
- Produces: `OpenAiAgentBackend` @ApplicationScoped implements AgentBackend with key `"openai"`. Config: `casehub.platform.agent.openai.{api-key, default-model, prompt-cache-retention, default-timeout, max-concurrent-sessions}`

- [ ] **Step 1: Create module POM**

Same structure as agent-claude/pom.xml. Dependencies: `agent-api`, `agent-router` (transitive), `quarkus-arc`, `com.openai:openai-java`.

**Note:** Verify the exact Maven coordinates for the OpenAI Java SDK before implementation. Search `mvn search` or `https://central.sonatype.com` for `com.openai:openai-java` to confirm the latest version.

- [ ] **Step 2: Add module to root POM**

Add `<module>agent-openai</module>` after `<module>agent-claude</module>`.

- [ ] **Step 3: Write failing test**

Test the backend via test constructor (stream factory pattern from ClaudeAgentClient):

```java
package io.casehub.platform.agent.openai;

import io.casehub.platform.agent.*;
import io.smallrye.mutiny.Multi;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class OpenAiAgentBackendTest {

    @Test
    void keyIsOpenai() {
        var backend = new OpenAiAgentBackend(testProperties(),
                config -> Multi.createFrom().item(new AgentEvent.TextDelta("hello")));
        assertThat(backend.key()).isEqualTo("openai");
    }

    @Test
    void invokeStreamsEvents() {
        var backend = new OpenAiAgentBackend(testProperties(),
                config -> Multi.createFrom().items(
                        new AgentEvent.TextDelta("hello "),
                        new AgentEvent.TextDelta("world")));
        var config = AgentSessionConfig.of("sys", "user", "openai");
        var events = backend.invoke(config).collect().asList().await().indefinitely();
        assertThat(events).hasSize(2);
    }

    @Test
    void semaphoreLimitReached() {
        var props = testProperties();
        when(props.maxConcurrentSessions()).thenReturn(1);
        var backend = new OpenAiAgentBackend(props,
                config -> Multi.createFrom().nothing());
        backend.invoke(AgentSessionConfig.of("sys", "user", "openai"));
        var result = backend.invoke(AgentSessionConfig.of("sys", "user", "openai"));
        assertThatThrownBy(() -> result.collect().asList().await().indefinitely())
                .isInstanceOf(AgentSessionLimitException.class);
    }

    static OpenAiAgentProperties testProperties() {
        var props = mock(OpenAiAgentProperties.class);
        when(props.defaultModel()).thenReturn("gpt-5.5");
        when(props.defaultTimeout()).thenReturn(java.time.Duration.ofMinutes(5));
        when(props.maxConcurrentSessions()).thenReturn(4);
        when(props.promptCacheRetention()).thenReturn("in_memory");
        return props;
    }
}
```

- [ ] **Step 4: Implement OpenAiAgentBackend**

Follow the same patterns as `ClaudeAgentClient`:
- Semaphore for concurrent session limiting
- ScheduledExecutorService for timeout
- Test constructor with stream factory
- @PostConstruct to validate API key
- Stream mapping: OpenAI streaming response → `AgentEvent` (TextDelta, ThinkingDelta, ToolCallComplete)
- Set `prompt_cache_key` and `prompt_cache_retention` on requests

```java
@ApplicationScoped
public class OpenAiAgentBackend implements AgentBackend {

    private final OpenAiAgentProperties properties;
    private final Semaphore semaphore;
    private final ScheduledExecutorService timeoutScheduler;
    private final Function<AgentSessionConfig, Multi<AgentEvent>> streamFactory;

    @Inject
    public OpenAiAgentBackend(OpenAiAgentProperties properties) {
        this.properties = properties;
        this.semaphore = new Semaphore(properties.maxConcurrentSessions());
        this.timeoutScheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "casehub-agent-openai-timeout");
            t.setDaemon(true);
            return t;
        });
        this.streamFactory = null;
    }

    // Test constructor
    public OpenAiAgentBackend(OpenAiAgentProperties properties,
                              Function<AgentSessionConfig, Multi<AgentEvent>> streamFactory) {
        this.properties = properties;
        this.semaphore = new Semaphore(properties.maxConcurrentSessions());
        this.timeoutScheduler = Executors.newSingleThreadScheduledExecutor(r -> {
            Thread t = new Thread(r, "casehub-agent-openai-timeout");
            t.setDaemon(true);
            return t;
        });
        this.streamFactory = streamFactory;
    }

    @Override public String key() { return "openai"; }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        if (!semaphore.tryAcquire()) {
            return Multi.createFrom().failure(
                    new AgentSessionLimitException(properties.maxConcurrentSessions()));
        }
        try {
            Multi<AgentEvent> stream = streamFactory != null
                    ? streamFactory.apply(config)
                    : buildEventStream(config);
            return stream
                    .runSubscriptionOn(Infrastructure.getDefaultWorkerPool())
                    .onCompletion().invoke(semaphore::release)
                    .onFailure().invoke(t -> semaphore.release())
                    .onCancellation().invoke(semaphore::release);
        } catch (Exception e) {
            semaphore.release();
            return Multi.createFrom().failure(e);
        }
    }

    @Override
    public AgentSession openSession(AgentSessionInit init) {
        // Multi-turn via OpenAI Responses API — follow ClaudeAgentSession pattern
        throw new UnsupportedOperationException("OpenAI multi-turn sessions not yet implemented");
    }
}
```

- [ ] **Step 5: Implement OpenAiAgentProperties**

```java
@ConfigMapping(prefix = "casehub.platform.agent.openai")
public interface OpenAiAgentProperties {
    Optional<String> apiKey();

    @WithDefault("gpt-5.5")
    String defaultModel();

    @WithDefault("in_memory")
    String promptCacheRetention();

    @WithDefault("PT5M")
    Duration defaultTimeout();

    @WithDefault("4")
    int maxConcurrentSessions();
}
```

- [ ] **Step 6: Run tests**

Run: `mvn -pl agent-openai test -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/116/platform add agent-openai/ pom.xml
git -C /Users/mdproctor/claude/casehub/slots/116/platform commit -m "feat: add agent-openai module — OpenAI API provider

Native OpenAI Java SDK integration with prompt_cache_key support.

Refs casehubio/devtown#186"
```

---

### Task 7: agent-codex/ — Codex CLI Provider

**Files:**
- Create: `agent-codex/pom.xml`
- Create: `agent-codex/src/main/java/io/casehub/platform/agent/codex/CodexAgentBackend.java`
- Create: `agent-codex/src/main/java/io/casehub/platform/agent/codex/CodexAgentProperties.java`
- Test: `agent-codex/src/test/java/io/casehub/platform/agent/codex/CodexAgentBackendTest.java`
- Modify: `pom.xml` (root — add module)

**Interfaces:**
- Consumes: `AgentBackend`, `AgentRuntime`, `AgentProcess` from Tasks 1-2
- Produces: `CodexAgentBackend` @ApplicationScoped implements AgentBackend with key `"codex"`. Injects `AgentRuntime`.

- [ ] **Step 1: Create module POM**

Dependencies: `agent-api`, `agent-router` (transitive), `agent-runtime` (transitive), `quarkus-arc`. No external SDK — subprocess I/O only.

- [ ] **Step 2: Add module to root POM**

- [ ] **Step 3: Write failing test**

Follow the same test constructor pattern. For CLI providers, the test constructor takes a `Function<AgentSessionConfig, Multi<AgentEvent>>` stream factory AND a mock `AgentRuntime`:

```java
@Test
void keyIsCodex() {
    var backend = new CodexAgentBackend(testProperties(),
            config -> Multi.createFrom().item(new AgentEvent.TextDelta("codex output")));
    assertThat(backend.key()).isEqualTo("codex");
}
```

- [ ] **Step 4: Implement CodexAgentBackend**

Same infrastructure patterns as ClaudeAgentClient (semaphore, timeout, test constructor). The production path spawns `codex` via `AgentRuntime`, parses stdout for JSON-line events, maps to `AgentEvent`.

```java
@ApplicationScoped
public class CodexAgentBackend implements AgentBackend {
    @Inject AgentRuntime runtime;
    @Inject CodexAgentProperties properties;

    // Test constructor bypasses @Inject
    public CodexAgentBackend(CodexAgentProperties properties,
                             Function<AgentSessionConfig, Multi<AgentEvent>> streamFactory) { ... }

    @Override public String key() { return "codex"; }

    @Override
    public Multi<AgentEvent> invoke(AgentSessionConfig config) {
        // Same semaphore + timeout pattern as OpenAiAgentBackend
        // Production: spawn via runtime.spawn(buildConfig(config))
        // Parse stdout JSON lines → AgentEvent
    }
}
```

- [ ] **Step 5: Implement CodexAgentProperties**

```java
@ConfigMapping(prefix = "casehub.platform.agent.codex")
public interface CodexAgentProperties {
    @WithDefault("codex")
    String binaryPath();

    @WithDefault("PT5M")
    Duration defaultTimeout();

    @WithDefault("4")
    int maxConcurrentSessions();
}
```

- [ ] **Step 6: Run tests and commit**

Same pattern as Task 6.

---

### Task 8: agent-gemini/ — Gemini API Provider

**Files:**
- Create: `agent-gemini/pom.xml`
- Create: `agent-gemini/src/main/java/io/casehub/platform/agent/gemini/GeminiAgentBackend.java`
- Create: `agent-gemini/src/main/java/io/casehub/platform/agent/gemini/GeminiAgentProperties.java`
- Test: `agent-gemini/src/test/java/io/casehub/platform/agent/gemini/GeminiAgentBackendTest.java`
- Modify: `pom.xml` (root — add module)

**Interfaces:**
- Consumes: `AgentBackend`, `AgentSessionConfig`, `AgentEvent` from Task 1
- Produces: `GeminiAgentBackend` @ApplicationScoped implements AgentBackend with key `"gemini"`. Config: `casehub.platform.agent.gemini.{api-key, default-model, cache-ttl, default-timeout, max-concurrent-sessions}`

Same structure as Task 6. Key differences:
- Uses Google GenAI SDK (`com.google.genai:google-genai`)
- Explicit caching: creates `cachedContent` on first call with same systemPrompt, reuses by ID
- Works around tools+caching incompatibility by separating cached context from tool declarations
- `@Override public String key() { return "gemini"; }`

- [ ] **Steps 1-7: Same pattern as Task 6**

Follow the same TDD cycle: POM, root POM, failing test, implement, properties, run tests, commit.

---

### Task 9: agent-gemini-cli/ — Gemini CLI Provider

**Files:**
- Create: `agent-gemini-cli/pom.xml`
- Create: `agent-gemini-cli/src/main/java/io/casehub/platform/agent/geminicli/GeminiCliAgentBackend.java`
- Create: `agent-gemini-cli/src/main/java/io/casehub/platform/agent/geminicli/GeminiCliAgentProperties.java`
- Test: `agent-gemini-cli/src/test/java/io/casehub/platform/agent/geminicli/GeminiCliAgentBackendTest.java`
- Modify: `pom.xml` (root — add module)

**Interfaces:**
- Consumes: `AgentBackend`, `AgentRuntime`, `AgentProcess` from Tasks 1-2
- Produces: `GeminiCliAgentBackend` @ApplicationScoped with key `"gemini-cli"`. Injects `AgentRuntime`.

Same structure as Task 7. Key differences:
- Spawns `gemini` binary instead of `codex`
- `@Override public String key() { return "gemini-cli"; }`

- [ ] **Steps 1-6: Same pattern as Task 7**

---

### Task 10: Integration — POM, BOM, Build Verification, CLAUDE.md

**Files:**
- Modify: `pom.xml` (root — verify all modules listed)
- Modify: `platform/src/main/java/io/casehub/platform/agent/NoOpAgentProvider.java` (log message)
- Modify: `CLAUDE.md` (module table, package structure)

**Interfaces:**
- Consumes: all previous tasks
- Produces: clean build across all modules

- [ ] **Step 1: Update NoOpAgentProvider log message**

```java
LOG.warn("NoOpAgentProvider is active — add casehub-platform-agent-router " +
         "with at least one backend (agent-claude, agent-openai, agent-codex, " +
         "agent-gemini, agent-gemini-cli, or agent-langchain4j) to the classpath");
```

- [ ] **Step 2: Verify root POM module ordering**

Ensure modules are in dependency order:
```xml
<module>agent-runtime</module>      <!-- after testing -->
<module>agent-claude</module>
<module>agent-openai</module>
<module>agent-codex</module>
<module>agent-gemini</module>
<module>agent-gemini-cli</module>
<module>agent-langchain4j</module>
<module>agent-router</module>       <!-- after all backends -->
<module>agent-gate</module>         <!-- after router -->
```

- [ ] **Step 3: Full build**

Run: `mvn --batch-mode install -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: BUILD SUCCESS

- [ ] **Step 4: Run all tests**

Run: `mvn --batch-mode test -f /Users/mdproctor/claude/casehub/slots/116/platform/pom.xml`
Expected: ALL PASS

- [ ] **Step 5: Update CLAUDE.md**

Add new modules to the module table and package structure sections. Update the agent-claude/ and agent-langchain4j/ entries to reflect the AgentBackend change.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/116/platform add pom.xml platform/ CLAUDE.md
git -C /Users/mdproctor/claude/casehub/slots/116/platform commit -m "chore: integration — POM ordering, NoOp log update, CLAUDE.md

Refs casehubio/devtown#186"
```
