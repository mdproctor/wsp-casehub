# @DefaultBean SPI No-Op Defaults Migration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Migrate all nine engine SPI no-op/empty default beans from bare `@ApplicationScoped` or `@Alternative @ApplicationScoped` to `@DefaultBean @ApplicationScoped` so they yield automatically to consumer implementations.

**Architecture:** Pure annotation change — no behavioural change to the engine's own code paths. The nine beans fall into two groups: four blocking beans gaining `@DefaultBean` (three bare `@ApplicationScoped` + `EmptyWorkerContextProvider`), and five reactive/diff beans swapping `@Alternative` for `@DefaultBean`. All live in `casehub-engine/runtime/src/main/java/io/casehub/engine/internal/`.

**Tech Stack:** Jakarta CDI (`jakarta.enterprise.inject.DefaultBean`), Quarkus ARC, JUnit 5 + AssertJ for unit tests.

**Issue:** casehubio/engine#257
**Protocol:** PP-20260514-engine-spi-noops-defaultbean

---

## Files Modified

All in `runtime/src/main/java/io/casehub/engine/internal/`:

| File | Change |
|------|--------|
| `worker/NoOpWorkerProvisioner.java` | Add `@DefaultBean` + import |
| `worker/NoOpCaseChannelProvider.java` | Add `@DefaultBean` + import |
| `worker/NoOpWorkerStatusListener.java` | Add `@DefaultBean` + import |
| `worker/EmptyWorkerContextProvider.java` | Add `@DefaultBean` + import |
| `diff/NoOpContextDiffStrategy.java` | Replace `@Alternative` with `@DefaultBean`, swap import, update Javadoc |
| `worker/NoOpReactiveWorkerProvisioner.java` | Replace `@Alternative` with `@DefaultBean`, swap import, update Javadoc |
| `worker/NoOpReactiveCaseChannelProvider.java` | Replace `@Alternative` with `@DefaultBean`, swap import, update Javadoc |
| `worker/NoOpReactiveWorkerStatusListener.java` | Replace `@Alternative` with `@DefaultBean`, swap import, update Javadoc |
| `worker/EmptyReactiveWorkerContextProvider.java` | Replace `@Alternative` with `@DefaultBean`, swap import, update Javadoc |

Protocol updated:
- `parent/docs/protocols/engine-spi-noops-defaultbean.md` — add `EmptyWorkerContextProvider` and `EmptyReactiveWorkerContextProvider` to "Beans to fix" table

---

## Task 1: Establish Green Baseline

Before changing anything, confirm the existing test suite passes.

**Files:** none modified

- [ ] **Step 1: Install deps and run runtime tests**

```bash
cd /Users/mdproctor/claude/casehub/engine
mvn install -DskipTests -q
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime
```

Expected: `BUILD SUCCESS` — all tests pass. If any fail, stop and investigate before proceeding.

---

## Task 2: Add @DefaultBean to Four Blocking Beans

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerProvisioner.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpCaseChannelProvider.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerStatusListener.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/EmptyWorkerContextProvider.java`

- [ ] **Step 1: Update NoOpWorkerProvisioner.java**

Replace the file content exactly:

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
package io.casehub.engine.internal.worker;

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.Worker;
import io.casehub.api.spi.ProvisioningException;
import io.casehub.api.spi.WorkerProvisioner;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import java.util.Set;

/**
 * Default no-op WorkerProvisioner that throws on every provision() call. Signals misconfiguration —
 * replace with a real implementation (e.g. Claudony's ClaudonyWorkerProvisioner) before
 * provisioning is needed.
 */
@DefaultBean
@ApplicationScoped
public class NoOpWorkerProvisioner implements WorkerProvisioner {

  @Override
  public Worker provision(Set<String> capabilities, ProvisionContext context) {
    throw new ProvisioningException(
        "No WorkerProvisioner configured — add an @ApplicationScoped WorkerProvisioner implementation");
  }

  @Override
  public void terminate(String workerId) {
    // intentional no-op — nothing to terminate
  }

  @Override
  public Set<String> getCapabilities() {
    return Set.of();
  }
}
```

- [ ] **Step 2: Update NoOpWorkerStatusListener.java**

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
package io.casehub.engine.internal.worker;

import io.casehub.api.model.WorkResult;
import io.casehub.api.spi.WorkerStatusListener;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import java.util.Map;

/** Default no-op WorkerStatusListener. Silently ignores all lifecycle events. */
@DefaultBean
@ApplicationScoped
public class NoOpWorkerStatusListener implements WorkerStatusListener {

  @Override
  public void onWorkerStarted(String workerId, Map<String, String> sessionMeta) {
    // intentional no-op
  }

  @Override
  public void onWorkerCompleted(String workerId, WorkResult result) {
    // intentional no-op
  }

  @Override
  public void onWorkerStalled(String workerId) {
    // intentional no-op
  }
}
```

- [ ] **Step 3: Update NoOpCaseChannelProvider.java**

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
package io.casehub.engine.internal.worker;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.qhorus.api.message.MessageType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Default no-op CaseChannelProvider. Returns sentinel channels with {@code backendType = "none"}.
 * All write operations are no-ops.
 */
@DefaultBean
@ApplicationScoped
public class NoOpCaseChannelProvider implements CaseChannelProvider {

  @Override
  public CaseChannel openChannel(UUID caseId, String purpose) {
    return new CaseChannel(caseId + "/" + purpose, purpose, purpose, "none", Map.of());
  }

  @Override
  public void postToChannel(CaseChannel channel, String from, String content, MessageType type) {
    // intentional no-op
  }

  @Override
  public void closeChannel(CaseChannel channel) {
    // intentional no-op
  }

  @Override
  public List<CaseChannel> listChannels(UUID caseId) {
    return List.of();
  }
}
```

- [ ] **Step 4: Update EmptyWorkerContextProvider.java**

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
package io.casehub.engine.internal.worker;

import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.CaseChannel;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.spi.CaseChannelProvider;
import io.casehub.api.spi.WorkerContextProvider;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Default WorkerContextProvider. Returns a minimal context with the task capability as the
 * description, prior-worker history omitted, and open channels populated from {@link
 * CaseChannelProvider#listChannels(UUID)}.
 */
@DefaultBean
@ApplicationScoped
public class EmptyWorkerContextProvider implements WorkerContextProvider {

  @Inject CaseChannelProvider caseChannelProvider; // package-private for test injection

  @Override
  public WorkerContext buildContext(String workerId, UUID caseId, WorkRequest task) {
    List<CaseChannel> channels =
        caseId != null ? caseChannelProvider.listChannels(caseId) : List.of();
    return new WorkerContext(
        task.capability(), caseId, channels, List.of(), PropagationContext.createRoot(), Map.of());
  }
}
```

- [ ] **Step 5: Run tests — must stay green**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime
```

Expected: `BUILD SUCCESS`

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerProvisioner.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpWorkerStatusListener.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpCaseChannelProvider.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/EmptyWorkerContextProvider.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix: @DefaultBean on blocking SPI no-op defaults (Refs #257)"
```

---

## Task 3: Migrate Five @Alternative Beans to @DefaultBean

**Files:**
- Modify: `runtime/src/main/java/io/casehub/engine/internal/diff/NoOpContextDiffStrategy.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerProvisioner.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveCaseChannelProvider.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerStatusListener.java`
- Modify: `runtime/src/main/java/io/casehub/engine/internal/worker/EmptyReactiveWorkerContextProvider.java`

- [ ] **Step 1: Update NoOpContextDiffStrategy.java**

Replace `@Alternative` with `@DefaultBean`, swap the import, and rewrite the Javadoc (the activation instructions via `selected-alternatives` no longer apply):

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
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;

/**
 * No-op {@link ContextDiffStrategy}: skips diff computation entirely. {@code contextChanges} is
 * omitted from {@code WORKER_EXECUTION_COMPLETED} metadata. Active by default — replace with a
 * real implementation to enable context diffing.
 */
@DefaultBean
@ApplicationScoped
public class NoOpContextDiffStrategy implements ContextDiffStrategy {

  @Override
  public JsonNode compute(JsonNode before, JsonNode after) {
    return null;
  }
}
```

- [ ] **Step 2: Update NoOpReactiveWorkerProvisioner.java**

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
package io.casehub.engine.internal.worker;

import io.casehub.api.model.ProvisionContext;
import io.casehub.api.model.Worker;
import io.casehub.api.spi.ProvisioningException;
import io.casehub.api.spi.ReactiveWorkerProvisioner;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import java.util.Set;

/**
 * Reactive no-op WorkerProvisioner. Fails {@code provision} with {@link ProvisioningException} to
 * signal misconfiguration. All other operations complete with no side-effects. Active by default —
 * replace with a real reactive implementation when the reactive pipeline is needed.
 */
@DefaultBean
@ApplicationScoped
public class NoOpReactiveWorkerProvisioner implements ReactiveWorkerProvisioner {

  @Override
  public Uni<Worker> provision(Set<String> capabilities, ProvisionContext context) {
    return Uni.createFrom()
        .failure(
            new ProvisioningException(
                "No ReactiveWorkerProvisioner configured — add an @ApplicationScoped"
                    + " ReactiveWorkerProvisioner implementation"));
  }

  @Override
  public Uni<Void> terminate(String workerId) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Set<String>> getCapabilities() {
    return Uni.createFrom().item(Set.of());
  }
}
```

- [ ] **Step 3: Update NoOpReactiveCaseChannelProvider.java**

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
package io.casehub.engine.internal.worker;

import io.casehub.api.model.CaseChannel;
import io.casehub.api.spi.ReactiveCaseChannelProvider;
import io.casehub.qhorus.api.message.MessageType;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Reactive no-op CaseChannelProvider. Returns sentinel channels with {@code backendType = "none"}.
 * All write operations complete with no side-effects. Active by default — replace with a real
 * reactive implementation when a channel backend is needed.
 */
@DefaultBean
@ApplicationScoped
public class NoOpReactiveCaseChannelProvider implements ReactiveCaseChannelProvider {

  @Override
  public Uni<CaseChannel> openChannel(UUID caseId, String purpose) {
    return Uni.createFrom()
        .item(new CaseChannel(caseId + "/" + purpose, purpose, purpose, "none", Map.of()));
  }

  @Override
  public Uni<Void> postToChannel(
      CaseChannel channel, String from, String content, MessageType type) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Void> closeChannel(CaseChannel channel) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<List<CaseChannel>> listChannels(UUID caseId) {
    return Uni.createFrom().item(List.of());
  }
}
```

- [ ] **Step 4: Update NoOpReactiveWorkerStatusListener.java**

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
package io.casehub.engine.internal.worker;

import io.casehub.api.model.WorkResult;
import io.casehub.api.spi.ReactiveWorkerStatusListener;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import java.util.Map;

/**
 * Reactive no-op WorkerStatusListener. Silently ignores all lifecycle events. Active by default —
 * replace with a real reactive implementation to observe worker lifecycle events.
 */
@DefaultBean
@ApplicationScoped
public class NoOpReactiveWorkerStatusListener implements ReactiveWorkerStatusListener {

  @Override
  public Uni<Void> onWorkerStarted(String workerId, Map<String, String> sessionMeta) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Void> onWorkerCompleted(String workerId, WorkResult result) {
    return Uni.createFrom().voidItem();
  }

  @Override
  public Uni<Void> onWorkerStalled(String workerId) {
    return Uni.createFrom().voidItem();
  }
}
```

- [ ] **Step 5: Update EmptyReactiveWorkerContextProvider.java**

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
package io.casehub.engine.internal.worker;

import io.casehub.api.context.PropagationContext;
import io.casehub.api.model.CaseChannel;
import io.casehub.api.model.WorkRequest;
import io.casehub.api.model.WorkerContext;
import io.casehub.api.spi.ReactiveCaseChannelProvider;
import io.casehub.api.spi.ReactiveWorkerContextProvider;
import io.smallrye.mutiny.Uni;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.DefaultBean;
import jakarta.inject.Inject;
import java.util.List;
import java.util.Map;
import java.util.UUID;

/**
 * Reactive default WorkerContextProvider. Returns a minimal context with the task capability as the
 * description, prior-worker history omitted, and open channels populated from {@link
 * ReactiveCaseChannelProvider#listChannels(UUID)}. Active by default — replace with a real
 * reactive implementation when richer context is needed.
 */
@DefaultBean
@ApplicationScoped
public class EmptyReactiveWorkerContextProvider implements ReactiveWorkerContextProvider {

  @Inject
  ReactiveCaseChannelProvider reactiveCaseChannelProvider; // package-private for test injection

  @Override
  public Uni<WorkerContext> buildContext(String workerId, UUID caseId, WorkRequest task) {
    Uni<List<CaseChannel>> channelsUni =
        caseId != null
            ? reactiveCaseChannelProvider.listChannels(caseId)
            : Uni.createFrom().item(List.of());
    return channelsUni.map(
        channels ->
            new WorkerContext(
                task.capability(),
                caseId,
                channels,
                List.of(),
                PropagationContext.createRoot(),
                Map.of()));
  }
}
```

- [ ] **Step 6: Run tests — must stay green**

```bash
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime
```

Expected: `BUILD SUCCESS`

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/engine add \
  runtime/src/main/java/io/casehub/engine/internal/diff/NoOpContextDiffStrategy.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerProvisioner.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveCaseChannelProvider.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/NoOpReactiveWorkerStatusListener.java \
  runtime/src/main/java/io/casehub/engine/internal/worker/EmptyReactiveWorkerContextProvider.java
git -C /Users/mdproctor/claude/casehub/engine commit -m "fix: @DefaultBean on @Alternative SPI no-op defaults (Refs #257)"
```

---

## Task 4: Update Protocol

The protocol `PP-20260514-engine-spi-noops-defaultbean` was written before `EmptyWorkerContextProvider` and `EmptyReactiveWorkerContextProvider` were identified. Update it to cover all nine beans.

**Files:**
- Modify: `parent/docs/protocols/engine-spi-noops-defaultbean.md`

- [ ] **Step 1: Replace the "Beans to fix" section**

The full updated protocol file:

```markdown
---
id: PP-20260514-engine-spi-noops-defaultbean
title: "Engine SPI no-op defaults must use @DefaultBean, not bare @ApplicationScoped"
type: rule
scope: platform
applies_to: "casehub-engine runtime — all no-op SPI default implementations"
severity: error
refs:
  - GE-20260428-9311f8
created: 2026-05-14
---

Every no-op SPI default in casehub-engine must be annotated `@DefaultBean @ApplicationScoped`,
not bare `@ApplicationScoped` or `@Alternative @ApplicationScoped`. A bare `@ApplicationScoped`
no-op collides with consumer implementations (e.g. Claudony's `ClaudonyWorkerProvisioner`) when
the engine runtime is indexed alongside a consumer repo's `@QuarkusTest` classpath, causing CDI
ambiguity errors that fail the entire test suite.

`@DefaultBean` yields automatically to any non-default qualifying bean — the correct semantic
for a platform fallback that exists only when no real implementation is provided.

## Beans

All in `casehub-engine/runtime/src/main/java/io/casehub/engine/internal/`:

| Class | Package | Annotation |
|-------|---------|-----------|
| `NoOpWorkerProvisioner` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `NoOpCaseChannelProvider` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `NoOpWorkerStatusListener` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `EmptyWorkerContextProvider` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `EmptyReactiveWorkerContextProvider` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `NoOpReactiveWorkerProvisioner` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `NoOpReactiveCaseChannelProvider` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `NoOpReactiveWorkerStatusListener` | `worker/` | `@DefaultBean @ApplicationScoped` |
| `NoOpContextDiffStrategy` | `diff/` | `@DefaultBean @ApplicationScoped` |

## Why this matters

Consumer repos (Claudony, devtown, etc.) that depend on `casehub-testing` for integration
tests index the engine runtime transitively. Without `@DefaultBean`, CDI sees two beans
implementing the same SPI interface — the engine's no-op and the consumer's real implementation
— and fails with an unsatisfied/ambiguous dependency error. The workaround (listing each no-op
in `quarkus.arc.exclude-types` per consumer) is fragile: it breaks silently when new no-ops are
added to the engine.

## Adding a new no-op default

When adding a new SPI no-op to `casehub-engine`:
1. Annotate it `@DefaultBean @ApplicationScoped` from the start
2. Add it to the table above
3. Verify no consumer repo needs updating (no `exclude-types` entries to clean up)

## See also

Garden entry `GE-20260428-9311f8` — "@ApplicationScoped no-op SPI beans collide with consumer
implementations when engine is indexed".
```

- [ ] **Step 2: Commit to parent repo**

```bash
git -C /Users/mdproctor/claude/casehub/parent add \
  docs/protocols/engine-spi-noops-defaultbean.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "protocol: expand engine-spi-noops-defaultbean to cover all nine beans (engine#257)"
```

---

## Task 5: Final Verification

- [ ] **Step 1: Full runtime test run**

```bash
cd /Users/mdproctor/claude/casehub/engine
TESTCONTAINERS_RYUK_DISABLED=true mvn test -pl runtime
```

Expected: `BUILD SUCCESS` — no failures, no regressions.

- [ ] **Step 2: Confirm all nine beans have @DefaultBean**

```bash
grep -rn "@DefaultBean" \
  /Users/mdproctor/claude/casehub/engine/runtime/src/main/java/io/casehub/engine/internal/
```

Expected: exactly 9 occurrences, one per bean listed in the Files Modified section above.

- [ ] **Step 3: Confirm no @Alternative remains in these files**

```bash
grep -rn "@Alternative" \
  /Users/mdproctor/claude/casehub/engine/runtime/src/main/java/io/casehub/engine/internal/
```

Expected: no output (zero occurrences).
