# ContextDiffStrategy Config-Driven Selection — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace inconsistent CDI annotations on the three `ContextDiffStrategy` implementations with a single config-driven `@Produces @DefaultBean` producer, making `none` the default and preserving consumer extensibility.

**Architecture:** A new `ContextDiffStrategyProducer` reads `casehub.engine.diff-strategy` (default `none`) and produces the chosen built-in implementation as a `@DefaultBean @ApplicationScoped` bean. The three strategy classes become plain POJOs with no CDI annotations. A consumer-provided `@ApplicationScoped ContextDiffStrategy` automatically wins over the default-bean produced instance.

**Tech Stack:** Java 17+, Quarkus ARC (`io.quarkus.arc.DefaultBean`), MicroProfile Config (`@ConfigProperty`), JUnit 5, AssertJ.

---

## File Map

| File | Action |
|------|--------|
| `api/src/test/java/io/casehub/api/spi/ContextDiffStrategyContractTest.java` | CREATE |
| `runtime/src/test/java/io/casehub/engine/internal/diff/ContextDiffStrategyProducerTest.java` | CREATE |
| `runtime/src/main/java/io/casehub/engine/internal/diff/ContextDiffStrategyProducer.java` | CREATE |
| `runtime/src/main/java/io/casehub/engine/internal/diff/NoOpContextDiffStrategy.java` | MODIFY — remove `@DefaultBean @ApplicationScoped` and unused imports |
| `runtime/src/main/java/io/casehub/engine/internal/diff/TopLevelContextDiffStrategy.java` | MODIFY — remove `@ApplicationScoped` and unused import |
| `runtime/src/main/java/io/casehub/engine/internal/diff/JsonPatchContextDiffStrategy.java` | MODIFY — remove `@Alternative @ApplicationScoped` and unused imports; update javadoc |
| `api/src/main/java/io/casehub/api/spi/ContextDiffStrategy.java` | MODIFY — update javadoc |
| `runtime/src/test/resources/application.properties` | MODIFY — add `casehub.engine.diff-strategy=top-level` |
| `docs/protocols/casehub/engine-spi-noops-defaultbean.md` | MODIFY — remove NoOp from table, add producer pattern note |

---

## Task 1: SPI Contract Test

**Files:**
- Create: `api/src/test/java/io/casehub/api/spi/ContextDiffStrategyContractTest.java`

- [ ] **Step 1.1: Write the contract test**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.api.spi;

import static org.assertj.core.api.Assertions.assertThat;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.junit.jupiter.api.Test;

class ContextDiffStrategyContractTest {

  private static final ObjectMapper MAPPER = new ObjectMapper();

  @Test
  void interface_hasComputeMethod() throws Exception {
    assertThat(
            ContextDiffStrategy.class.getMethod("compute", JsonNode.class, JsonNode.class))
        .isNotNull();
  }

  @Test
  void compute_returningNull_isValidNoOpContract() {
    ContextDiffStrategy noOp = (before, after) -> null;
    JsonNode node = MAPPER.createObjectNode();
    assertThat(noOp.compute(node, node)).isNull();
  }

  @Test
  void compute_canReturnNonNull() {
    ContextDiffStrategy passThrough = (before, after) -> after;
    JsonNode node = MAPPER.createObjectNode();
    assertThat(passThrough.compute(node, node)).isSameAs(node);
  }
}
```

- [ ] **Step 1.2: Run the test**

```bash
mvn install -DskipTests -q && mvn test -pl api -Dtest=ContextDiffStrategyContractTest
```

Expected: **PASS** — the interface already exists and the contract is trivially satisfied.

- [ ] **Step 1.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add api/src/test/java/io/casehub/api/spi/ContextDiffStrategyContractTest.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "test(api): ContextDiffStrategy SPI contract test (Refs #258)"
```

---

## Task 2: Producer Unit Tests — RED

**Files:**
- Create: `runtime/src/test/java/io/casehub/engine/internal/diff/ContextDiffStrategyProducerTest.java`

- [ ] **Step 2.1: Write the producer tests**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.diff;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

import org.junit.jupiter.api.Test;

class ContextDiffStrategyProducerTest {

  private ContextDiffStrategyProducer producer(String strategy) {
    ContextDiffStrategyProducer p = new ContextDiffStrategyProducer();
    p.strategy = strategy;
    return p;
  }

  @Test
  void none_producesNoOp() {
    assertThat(producer("none").produce()).isInstanceOf(NoOpContextDiffStrategy.class);
  }

  @Test
  void topLevel_producesTopLevel() {
    assertThat(producer("top-level").produce()).isInstanceOf(TopLevelContextDiffStrategy.class);
  }

  @Test
  void jsonPatch_producesJsonPatch() {
    assertThat(producer("json-patch").produce()).isInstanceOf(JsonPatchContextDiffStrategy.class);
  }

  @Test
  void unknownStrategy_throwsIllegalStateException() {
    assertThatThrownBy(() -> producer("bogus").produce())
        .isInstanceOf(IllegalStateException.class)
        .hasMessageContaining("casehub.engine.diff-strategy")
        .hasMessageContaining("bogus");
  }
}
```

- [ ] **Step 2.2: Run to confirm RED**

```bash
mvn install -DskipTests -q && TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=ContextDiffStrategyProducerTest
```

Expected: **FAIL** — `ContextDiffStrategyProducer` does not exist yet.

---

## Task 3: Implement the Producer — GREEN

**Files:**
- Create: `runtime/src/main/java/io/casehub/engine/internal/diff/ContextDiffStrategyProducer.java`

- [ ] **Step 3.1: Create the producer**

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.diff;

import io.casehub.api.spi.ContextDiffStrategy;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import org.eclipse.microprofile.config.inject.ConfigProperty;

@ApplicationScoped
class ContextDiffStrategyProducer {

  @ConfigProperty(name = "casehub.engine.diff-strategy", defaultValue = "none")
  String strategy;

  @Produces
  @DefaultBean
  @ApplicationScoped
  ContextDiffStrategy produce() {
    return switch (strategy) {
      case "none" -> new NoOpContextDiffStrategy();
      case "top-level" -> new TopLevelContextDiffStrategy();
      case "json-patch" -> new JsonPatchContextDiffStrategy();
      default ->
          throw new IllegalStateException(
              "Unknown casehub.engine.diff-strategy: '"
                  + strategy
                  + "'. Valid values: none, top-level, json-patch");
    };
  }
}
```

- [ ] **Step 3.2: Run producer tests — confirm GREEN**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime -Dtest=ContextDiffStrategyProducerTest
```

Expected: **PASS** — all four producer tests pass.

---

## Task 4: Strip CDI Annotations + Fix Test Config

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/diff/NoOpContextDiffStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/diff/TopLevelContextDiffStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/diff/JsonPatchContextDiffStrategy.java`
- Modify: `runtime/src/test/resources/application.properties`

- [ ] **Step 4.1: Strip `NoOpContextDiffStrategy`**

Remove the `@DefaultBean` annotation, `@ApplicationScoped` annotation, and their imports. The class becomes a plain POJO. Replace the full class body with:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.diff;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.spi.ContextDiffStrategy;

/**
 * No-op {@link ContextDiffStrategy}: skips diff computation entirely. {@code contextChanges} is
 * omitted from {@code WORKER_EXECUTION_COMPLETED} metadata.
 *
 * <p>Active when {@code casehub.engine.diff-strategy=none} (the default).
 */
class NoOpContextDiffStrategy implements ContextDiffStrategy {

  @Override
  public JsonNode compute(JsonNode before, JsonNode after) {
    return null;
  }
}
```

- [ ] **Step 4.2: Strip `TopLevelContextDiffStrategy`**

Remove `@ApplicationScoped` annotation and its import. Also drop the `public` modifier — it was required for CDI proxying and is unnecessary for a package-private POJO in `internal.diff`. The method body is unchanged; only the class declaration and imports change:

```java
// Remove this import:
import jakarta.enterprise.context.ApplicationScoped;

// Remove this annotation from the class:
@ApplicationScoped

// Change class declaration from:
public class TopLevelContextDiffStrategy implements ContextDiffStrategy {
// To:
class TopLevelContextDiffStrategy implements ContextDiffStrategy {
```

- [ ] **Step 4.3: Strip `JsonPatchContextDiffStrategy`**

Remove `@Alternative`, `@ApplicationScoped`, and their imports. Update the javadoc to remove the "Activate via: quarkus.arc.selected-alternatives" block. Replace the full class body with:

```java
/*
 * Copyright 2026-Present The Case Hub Authors
 *
 * Licensed under the Apache License, Version 2.0 (the "License");
 * you may not use this file except in compliance with the License.
 * You may obtain a copy of the License at
 *
 * http://www.apache.org/licenses/LICENSE-2.0
 *
 * Unless required by applicable law or agreed to in writing, software
 * distributed under the License is distributed on an "AS IS" BASIS,
 * WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied.
 * See the License for the specific language governing permissions and
 * limitations under the License.
 */
package io.casehub.engine.internal.diff;

import com.fasterxml.jackson.databind.JsonNode;
import io.casehub.api.spi.ContextDiffStrategy;
import io.fabric8.zjsonpatch.JsonDiff;

/**
 * RFC 6902 JSON Patch {@link ContextDiffStrategy} using {@code zjsonpatch}.
 *
 * <p>Produces a JSON array of patch operations ({@code add}, {@code replace}, {@code remove}).
 * Records changes at every affected path, including nested keys — more precise than {@link
 * TopLevelContextDiffStrategy} but less readable for humans.
 *
 * <p>Active when {@code casehub.engine.diff-strategy=json-patch}.
 *
 * <p>Useful as a foundation for future replay support (issues #10–#13).
 */
class JsonPatchContextDiffStrategy implements ContextDiffStrategy {

  @Override
  public JsonNode compute(JsonNode before, JsonNode after) {
    return JsonDiff.asJson(before, after);
  }
}
```

- [ ] **Step 4.4: Set explicit diff strategy in test properties**

The `ContextDiffEndToEndTest` tests the `TopLevelContextDiffStrategy` behaviour. Now that `none` is the default, add an explicit config entry to `runtime/src/test/resources/application.properties`:

```properties
# Context diff strategy for end-to-end tests — explicit top-level (none is the default)
casehub.engine.diff-strategy=top-level
```

- [ ] **Step 4.5: Run all runtime tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime
```

Expected: **all tests PASS** — producer tests, strategy unit tests, and `ContextDiffEndToEndTest` all green.

- [ ] **Step 4.6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/diff/ContextDiffStrategyProducer.java \
  runtime/src/main/java/io/casehub/engine/internal/diff/NoOpContextDiffStrategy.java \
  runtime/src/main/java/io/casehub/engine/internal/diff/TopLevelContextDiffStrategy.java \
  runtime/src/main/java/io/casehub/engine/internal/diff/JsonPatchContextDiffStrategy.java \
  runtime/src/test/java/io/casehub/engine/internal/diff/ContextDiffStrategyProducerTest.java \
  runtime/src/test/resources/application.properties
git -C /Users/mdproctor/claude/casehub/engine commit -m "refactor(diff): config-driven ContextDiffStrategy producer, strip CDI from strategy classes (Refs #258)"
```

---

## Task 5: Javadoc + Protocol Updates

**Files:**
- Modify: `api/src/main/java/io/casehub/api/spi/ContextDiffStrategy.java`
- Modify: `docs/protocols/casehub/engine-spi-noops-defaultbean.md`

- [ ] **Step 5.1: Update `ContextDiffStrategy` javadoc**

Replace the existing class-level javadoc with:

```java
/**
 * SPI: computes the diff between CaseContext state before and after a worker execution.
 *
 * <p>The result is stored as {@code contextChanges} in the {@code WORKER_EXECUTION_COMPLETED}
 * EventLog metadata. Return {@code null} to omit {@code contextChanges} entirely.
 *
 * <p>Select a built-in implementation via:
 *
 * <pre>casehub.engine.diff-strategy=none|top-level|json-patch</pre>
 *
 * <ul>
 *   <li>{@code none} (default) — returns null; no diff overhead
 *   <li>{@code top-level} — per-key before/after object
 *   <li>{@code json-patch} — RFC 6902 patch array
 * </ul>
 *
 * <p>A consumer {@code @ApplicationScoped} implementation of this interface takes priority over
 * the config-selected built-in.
 */
```

- [ ] **Step 5.2: Update `engine-spi-noops-defaultbean.md`**

Remove `NoOpContextDiffStrategy` from the beans table. Add a section after the table distinguishing the two patterns.

In the `## Beans` section, remove this row:
```
| `NoOpContextDiffStrategy` | `diff/` | `@DefaultBean @ApplicationScoped` |
```

Add a new section after the existing `## Why this matters` section:

```markdown
## Two patterns: consumer-replaceable SPI vs. engine-internal selection

`@DefaultBean` applies to two distinct situations in the engine:

**Consumer-replaceable SPI** (the 8 worker/channel beans above): The engine ships a no-op
fallback. A consumer deployment provides a real `@ApplicationScoped` implementation
(e.g. `ClaudonyWorkerProvisioner`) and the no-op yields automatically.

**Engine-internal strategy selection** (`ContextDiffStrategy`): The engine ships multiple
real implementations and selects one via config (`casehub.engine.diff-strategy`). A
`@Produces @DefaultBean @ApplicationScoped` method on `ContextDiffStrategyProducer` produces
the chosen instance. A consumer `@ApplicationScoped` implementation still wins over the
produced default. The individual strategy classes (`NoOpContextDiffStrategy` etc.) are plain
POJOs — no CDI annotations — instantiated directly by the producer.

Do not apply the single-class `@DefaultBean` pattern to engine-internal strategy groups.
Use a config-driven producer instead.
```

- [ ] **Step 5.3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  api/src/main/java/io/casehub/api/spi/ContextDiffStrategy.java
git -C /Users/mdproctor/claude/casehub/parent add \
  docs/protocols/casehub/engine-spi-noops-defaultbean.md
git -C /Users/mdproctor/claude/casehub/engine commit -m "docs(api): update ContextDiffStrategy javadoc — config property and consumer override (Refs #258)"
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(protocol): distinguish consumer-replaceable SPI from engine-internal strategy selection (Refs casehubio/engine#258)"
```

---

## Task 6: Full Test Run

- [ ] **Step 6.1: Install all modules**

```bash
mvn install -DskipTests -q
```

- [ ] **Step 6.2: Run api tests**

```bash
mvn test -pl api
```

Expected: all pass including `ContextDiffStrategyContractTest`.

- [ ] **Step 6.3: Run runtime tests**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime
```

Expected: all pass — strategy unit tests, producer unit tests, `ContextDiffEndToEndTest`.

---

## Task 7: Create Follow-up Issue

The E2E verification of the `none` default is out of scope for this PR (requires a separate `@QuarkusTestProfile` and second Quarkus context startup). Create the issue before closing #258.

- [ ] **Step 7.1: Create the follow-up issue**

```bash
gh issue create -R casehubio/engine \
  --title "test(diff): E2E verification that none default omits contextChanges" \
  --body "$(cat <<'EOF'
## Context

engine#258 introduced \`casehub.engine.diff-strategy\` with \`none\` as the default. The unit-level \`ContextDiffStrategyProducerTest\` verifies the \`none\` case returns a \`NoOpContextDiffStrategy\`. An end-to-end test verifying that \`WORKER_EXECUTION_COMPLETED\` EventLog entries have **no** \`contextChanges\` field when the default is active is missing.

## Task

Add a \`@QuarkusTest\` (separate \`@QuarkusTestProfile\` with no \`casehub.engine.diff-strategy\` set, or explicitly \`none\`) that:
1. Runs a case with a worker that modifies context
2. Asserts the resulting EventLog entry has **no** \`contextChanges\` field

## Why deferred

Requires a separate \`@QuarkusTestProfile\` and second Quarkus application context startup, which is out of scope for the engine#258 refactor PR.
EOF
)"
```
