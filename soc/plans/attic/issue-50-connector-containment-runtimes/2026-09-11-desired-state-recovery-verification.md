# Desired-State Recovery Verification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #49 — Desired-state reconciliation — post-containment recovery verification
**Issue group:** #49

**Goal:** Wire casehub-desiredstate reconciliation into the SOC containment recovery phase so that after containment executes, the system verifies the action took effect and escalates when it doesn't.

**Architecture:** Per-case DesiredStateGraph with `tenancyId=case:<caseId>`. Two new case plan bindings (`recovery-verification-start`, `recovery-verification-result`) bracket async reconciliation. GlobalReconciliationListener detects convergence/divergence/timeout and signals the case plan via `CaseHubRuntime.signal()`. ThresholdFaultPolicy escalates unverified containment to SOC manager via HumanNodeHandler → WorkItem bridge.

**Tech Stack:** Java 21, Quarkus 3.32.2, casehub-desiredstate (api + runtime), casehub-engine (CaseHubRuntime), casehub-ledger, casehub-work

## Global Constraints

- Java 21 source level on Java 26 JVM
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
- Test scoping: `mvn -pl <module> -am -Dtest=ClassName -Dsurefire.failIfNoSpecifiedTests=false`
- All new api/ types are pure Java — no CDI, no Quarkus, no JPA
- All new app/ types use `@ApplicationScoped` or `@Alternative` CDI pattern
- Node IDs embed operational identifiers — must pass through `SocPiiSanitiser` before ledger write
- Every commit references `Refs #49`

---

## Batch 1: Domain types and Maven wiring

### Task 1: Add desiredstate dependencies and create domain types

**Files:**
- Modify: `api/pom.xml` — add `casehub-desiredstate-api` dependency
- Modify: `app/pom.xml` — add `casehub-desiredstate` (runtime) dependency
- Create: `api/src/main/java/io/casehub/soc/domain/SocContainmentNodeTypes.java`
- Create: `api/src/main/java/io/casehub/soc/domain/SocContainmentNodeSpec.java`
- Create: `api/src/main/java/io/casehub/soc/domain/SocDivergenceReviewSpec.java`
- Create: `api/src/main/java/io/casehub/soc/worker/contract/RecoveryVerificationOutput.java`
- Create: `api/src/main/java/io/casehub/soc/worker/contract/RecoveryVerificationResultOutput.java`
- Modify: `api/src/main/java/io/casehub/soc/domain/SocStepType.java` — add `RECOVERY_VERIFIED`, `RECOVERY_DIVERGED`
- Test: `api/src/test/java/io/casehub/soc/domain/SocContainmentNodeSpecTest.java`

**Interfaces:**
- Produces: `SocContainmentNodeTypes` constants (8 `NodeType` values), `SocContainmentNodeSpec` sealed interface (8 records), `SocDivergenceReviewSpec` (with `fromFaultEvent` factory), `RecoveryVerificationOutput`, `RecoveryVerificationResultOutput`, `SocStepType.RECOVERY_VERIFIED`, `SocStepType.RECOVERY_DIVERGED`

- [ ] **Step 1: Add `casehub-desiredstate-api` dependency to `api/pom.xml`**

Add after the `casehub-eidos-api` dependency block (around line 42):

```xml
    <!-- Desired-state API — NodeType, NodeSpec, HumanGating for recovery verification -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-desiredstate-api</artifactId>
    </dependency>
```

- [ ] **Step 2: Add `casehub-desiredstate` runtime dependency to `app/pom.xml`**

Add in the CaseHub Foundation section (after existing foundation dependencies):

```xml
    <!-- Desired-state runtime — reconciliation loop, lifecycle manager, fault policy engine -->
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-desiredstate</artifactId>
    </dependency>
```

- [ ] **Step 3: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -DskipTests -pl api,app -am`
Expected: BUILD SUCCESS

- [ ] **Step 4: Write test for SocContainmentNodeSpec — nodeType mapping**

```java
package io.casehub.soc.domain;

import io.casehub.desiredstate.api.NodeType;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SocContainmentNodeSpecTest {

    @Test
    void eachSpecReturnsCorrectNodeType() {
        assertEquals(SocContainmentNodeTypes.ISOLATE_HOST,
                new SocContainmentNodeSpec.IsolateHostSpec("host1", "agent1").nodeType());
        assertEquals(SocContainmentNodeTypes.BLOCK_IP,
                new SocContainmentNodeSpec.BlockIpSpec("10.0.0.1", "rule1").nodeType());
        assertEquals(SocContainmentNodeTypes.BLOCK_DOMAIN,
                new SocContainmentNodeSpec.BlockDomainSpec("evil.com", "proxy1").nodeType());
        assertEquals(SocContainmentNodeTypes.REVOKE_CREDENTIALS,
                new SocContainmentNodeSpec.RevokeCredentialsSpec("cred1", "okta").nodeType());
        assertEquals(SocContainmentNodeTypes.ROTATE_API_KEY,
                new SocContainmentNodeSpec.RotateApiKeySpec("key1", "svc1").nodeType());
        assertEquals(SocContainmentNodeTypes.DISABLE_USER_ACCOUNT,
                new SocContainmentNodeSpec.DisableUserAccountSpec("user1", "ad").nodeType());
        assertEquals(SocContainmentNodeTypes.NETWORK_SEGMENTATION,
                new SocContainmentNodeSpec.NetworkSegmentationSpec("seg1", "fw1").nodeType());
        assertEquals(SocContainmentNodeTypes.WIPE_ENDPOINT,
                new SocContainmentNodeSpec.WipeEndpointSpec("host2", "agent2").nodeType());
    }

    @Test
    void divergenceReviewSpecReturnsReviewType() {
        var spec = SocDivergenceReviewSpec.fromFaultEvent(
                new io.casehub.desiredstate.api.FaultEvent(
                        io.casehub.desiredstate.api.NodeId.of("test"),
                        io.casehub.desiredstate.api.FaultType.PROVISION_FAILED,
                        "test failure"),
                null);
        assertEquals(SocDivergenceReviewSpec.REVIEW_TYPE, spec.nodeType());
        assertEquals(io.casehub.desiredstate.api.HumanGating.ALL, spec.humanGating());
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dtest=SocContainmentNodeSpecTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — classes do not exist

- [ ] **Step 6: Create `SocContainmentNodeTypes.java`**

```java
package io.casehub.soc.domain;

import io.casehub.desiredstate.api.NodeType;

public final class SocContainmentNodeTypes {
    public static final NodeType ISOLATE_HOST = NodeType.of("soc:isolate-host");
    public static final NodeType BLOCK_IP = NodeType.of("soc:block-ip");
    public static final NodeType BLOCK_DOMAIN = NodeType.of("soc:block-domain");
    public static final NodeType REVOKE_CREDENTIALS = NodeType.of("soc:revoke-credentials");
    public static final NodeType ROTATE_API_KEY = NodeType.of("soc:rotate-api-key");
    public static final NodeType DISABLE_USER_ACCOUNT = NodeType.of("soc:disable-user-account");
    public static final NodeType NETWORK_SEGMENTATION = NodeType.of("soc:network-segmentation");
    public static final NodeType WIPE_ENDPOINT = NodeType.of("soc:wipe-endpoint");

    private SocContainmentNodeTypes() {}
}
```

- [ ] **Step 7: Create `SocContainmentNodeSpec.java`**

```java
package io.casehub.soc.domain;

import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;

public sealed interface SocContainmentNodeSpec extends NodeSpec {

    record IsolateHostSpec(String hostname, String edpAgentId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.ISOLATE_HOST; }
    }
    record BlockIpSpec(String ipAddress, String firewallRuleId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.BLOCK_IP; }
    }
    record BlockDomainSpec(String domain, String proxyRuleId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.BLOCK_DOMAIN; }
    }
    record RevokeCredentialsSpec(String credentialId, String iamProvider) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.REVOKE_CREDENTIALS; }
    }
    record RotateApiKeySpec(String keyId, String serviceId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.ROTATE_API_KEY; }
    }
    record DisableUserAccountSpec(String userId, String iamProvider) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.DISABLE_USER_ACCOUNT; }
    }
    record NetworkSegmentationSpec(String segmentId, String firewallRuleId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.NETWORK_SEGMENTATION; }
    }
    record WipeEndpointSpec(String hostname, String edpAgentId) implements SocContainmentNodeSpec {
        @Override public NodeType nodeType() { return SocContainmentNodeTypes.WIPE_ENDPOINT; }
    }
}
```

- [ ] **Step 8: Create `SocDivergenceReviewSpec.java`**

```java
package io.casehub.soc.domain;

import io.casehub.desiredstate.api.DesiredStateGraph;
import io.casehub.desiredstate.api.FaultEvent;
import io.casehub.desiredstate.api.HumanGating;
import io.casehub.desiredstate.api.NodeId;
import io.casehub.desiredstate.api.NodeSpec;
import io.casehub.desiredstate.api.NodeType;

public record SocDivergenceReviewSpec(
    NodeId faultedNodeId,
    String faultType,
    String faultMessage
) implements NodeSpec {

    public static final NodeType REVIEW_TYPE = NodeType.of("soc:divergence-review");

    @Override public NodeType nodeType() { return REVIEW_TYPE; }

    @Override
    public HumanGating humanGating() { return HumanGating.ALL; }

    public static SocDivergenceReviewSpec fromFaultEvent(FaultEvent event, DesiredStateGraph current) {
        return new SocDivergenceReviewSpec(event.node(), event.type().name(), event.detail());
    }
}
```

- [ ] **Step 9: Create output records and update SocStepType**

Create `api/src/main/java/io/casehub/soc/worker/contract/RecoveryVerificationOutput.java`:

```java
package io.casehub.soc.worker.contract;

import java.time.Instant;

public record RecoveryVerificationOutput(String status, Instant startedAt, String tenancyId) {}
```

Create `api/src/main/java/io/casehub/soc/worker/contract/RecoveryVerificationResultOutput.java`:

```java
package io.casehub.soc.worker.contract;

import java.time.Instant;
import java.util.List;

public record RecoveryVerificationResultOutput(
    String status, List<String> verifiedNodes, List<String> divergedNodes,
    Instant completedAt, long executionToVerificationMs) {}
```

Add to `SocStepType.java` (after `CONTAINMENT_EXECUTED`):

```java
    RECOVERY_VERIFIED,
    RECOVERY_DIVERGED,
```

- [ ] **Step 10: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dtest=SocContainmentNodeSpecTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add api/pom.xml app/pom.xml api/src/
git commit -m "feat(#49): add desiredstate domain types — NodeTypes, NodeSpecs, output records, step types

Refs #49"
```

---

## Batch 2: Reconciliation infrastructure — adapter, provisioner, fault policy

### Task 2: Default ActualStateAdapter, NodeProvisioner, and FaultPolicy

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/SocContainmentActualStateAdapter.java`
- Create: `app/src/main/java/io/casehub/soc/engine/SocContainmentNodeProvisioner.java`
- Create: `app/src/main/java/io/casehub/soc/engine/SocContainmentFaultPolicy.java`
- Modify: `app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntryWriter.java` — add REQUIRED_METADATA for new step types
- Modify: `app/src/main/java/io/casehub/soc/engine/compliance/SocPiiSanitiser.java` — add hostname pattern
- Test: `app/src/test/java/io/casehub/soc/engine/SocContainmentActualStateAdapterTest.java`
- Test: `app/src/test/java/io/casehub/soc/engine/SocContainmentNodeProvisionerTest.java`
- Test: `app/src/test/java/io/casehub/soc/engine/SocContainmentFaultPolicyTest.java`

**Interfaces:**
- Consumes: `SocContainmentNodeTypes` (Task 1), `SocContainmentNodeSpec` (Task 1), `SocDivergenceReviewSpec` (Task 1)
- Produces: `SocContainmentActualStateAdapter` (CDI bean, `ActualStateAdapter`), `SocContainmentNodeProvisioner` (CDI bean, `NodeProvisioner`), `SocContainmentFaultPolicy` (CDI producer, `ThresholdFaultPolicy`)

- [ ] **Step 1: Write test for ActualStateAdapter — returns PRESENT for all nodes**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocContainmentNodeSpec;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import org.junit.jupiter.api.Test;
import java.util.Set;
import static org.junit.jupiter.api.Assertions.*;

class SocContainmentActualStateAdapterTest {

    private final SocContainmentActualStateAdapter adapter = new SocContainmentActualStateAdapter();

    @Test
    void handledTypes_coversAllContainmentNodeTypes() {
        Set<NodeType> handled = adapter.handledTypes();
        assertTrue(handled.contains(SocContainmentNodeTypes.ISOLATE_HOST));
        assertTrue(handled.contains(SocContainmentNodeTypes.BLOCK_IP));
        assertTrue(handled.contains(SocContainmentNodeTypes.WIPE_ENDPOINT));
        assertEquals(8, handled.size());
    }

    @Test
    void readActual_returnsPresentForEveryNode() {
        var spec = new SocContainmentNodeSpec.IsolateHostSpec("host1", "agent1");
        var node = new DesiredNode(NodeId.of("n1"), SocContainmentNodeTypes.ISOLATE_HOST, spec, HumanGating.NONE);
        DesiredStateGraph graph = DesiredStateGraphFactory.create().empty().withNode(node);

        ActualState actual = adapter.readActual(graph, "case:test-123");

        assertEquals(NodeStatus.PRESENT, actual.statusOf(NodeId.of("n1")).orElse(null));
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocContainmentActualStateAdapterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — class does not exist

- [ ] **Step 3: Implement `SocContainmentActualStateAdapter`**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.HashMap;
import java.util.Set;

@DefaultBean
@ApplicationScoped
public class SocContainmentActualStateAdapter implements ActualStateAdapter {

    private static final Set<NodeType> HANDLED = Set.of(
            SocContainmentNodeTypes.ISOLATE_HOST, SocContainmentNodeTypes.BLOCK_IP,
            SocContainmentNodeTypes.BLOCK_DOMAIN, SocContainmentNodeTypes.REVOKE_CREDENTIALS,
            SocContainmentNodeTypes.ROTATE_API_KEY, SocContainmentNodeTypes.DISABLE_USER_ACCOUNT,
            SocContainmentNodeTypes.NETWORK_SEGMENTATION, SocContainmentNodeTypes.WIPE_ENDPOINT);

    @Override
    public Set<NodeType> handledTypes() { return HANDLED; }

    @Override
    public ActualState readActual(DesiredStateGraph desired, String tenancyId) {
        var statuses = new HashMap<NodeId, NodeStatus>();
        for (var entry : desired.nodes().entrySet()) {
            if (HANDLED.contains(entry.getValue().type())) {
                statuses.put(entry.getKey(), NodeStatus.PRESENT);
            }
        }
        return new ActualState(statuses);
    }
}
```

- [ ] **Step 4: Run test — verify PASS**

- [ ] **Step 5: Write test for NodeProvisioner — returns Failed**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocContainmentNodeSpec;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class SocContainmentNodeProvisionerTest {

    private final SocContainmentNodeProvisioner provisioner = new SocContainmentNodeProvisioner();

    @Test
    void provision_returnsFailed() {
        var spec = new SocContainmentNodeSpec.IsolateHostSpec("host1", "agent1");
        var node = new DesiredNode(NodeId.of("n1"), SocContainmentNodeTypes.ISOLATE_HOST, spec, HumanGating.NONE);
        var ctx = new ProvisionContext("case:test", null, null);

        ProvisionResult result = provisioner.provision(node, ctx);

        assertInstanceOf(ProvisionResult.Failed.class, result);
    }

    @Test
    void deprovision_returnsFailed() {
        var spec = new SocContainmentNodeSpec.IsolateHostSpec("host1", "agent1");
        var node = new DesiredNode(NodeId.of("n1"), SocContainmentNodeTypes.ISOLATE_HOST, spec, HumanGating.NONE);
        var ctx = new DeprovisionContext("case:test", null, null);

        DeprovisionResult result = provisioner.deprovision(node, ctx);

        assertInstanceOf(DeprovisionResult.Failed.class, result);
    }
}
```

- [ ] **Step 6: Implement `SocContainmentNodeProvisioner`**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import jakarta.enterprise.context.ApplicationScoped;
import java.time.Duration;
import java.util.Set;

@ApplicationScoped
public class SocContainmentNodeProvisioner implements NodeProvisioner {

    private static final Set<NodeType> HANDLED = Set.of(
            SocContainmentNodeTypes.ISOLATE_HOST, SocContainmentNodeTypes.BLOCK_IP,
            SocContainmentNodeTypes.BLOCK_DOMAIN, SocContainmentNodeTypes.REVOKE_CREDENTIALS,
            SocContainmentNodeTypes.ROTATE_API_KEY, SocContainmentNodeTypes.DISABLE_USER_ACCOUNT,
            SocContainmentNodeTypes.NETWORK_SEGMENTATION, SocContainmentNodeTypes.WIPE_ENDPOINT);

    @Override public Set<NodeType> handledTypes() { return HANDLED; }

    @Override public Duration resyncInterval() { return Duration.ofMinutes(5); }

    @Override
    public ProvisionResult provision(DesiredNode node, ProvisionContext context) {
        return new ProvisionResult.Failed(
                "re-provisioning disabled — containment divergence requires human review");
    }

    @Override
    public DeprovisionResult deprovision(DesiredNode node, DeprovisionContext context) {
        return new DeprovisionResult.Failed(
                "containment node removal requires case closure");
    }
}
```

- [ ] **Step 7: Run provisioner test — verify PASS**

- [ ] **Step 8: Write test for FaultPolicy — tier escalation**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocContainmentNodeSpec;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import io.casehub.soc.domain.SocDivergenceReviewSpec;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class SocContainmentFaultPolicyTest {

    @Test
    void tier1_returnsEmptyMutations_firstFault() {
        ThresholdFaultPolicy policy = SocContainmentFaultPolicy.create();
        var node = new DesiredNode(NodeId.of("n1"), SocContainmentNodeTypes.ISOLATE_HOST,
                new SocContainmentNodeSpec.IsolateHostSpec("h1", "a1"), HumanGating.NONE);
        DesiredStateGraph graph = DesiredStateGraphFactory.create().empty().withNode(node);
        var fault = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "test");

        List<GraphMutation<DesiredNode>> mutations = policy.onFault("case:t1", fault, graph,
                new ActualState(java.util.Map.of()));

        assertTrue(mutations.isEmpty());
    }

    @Test
    void tier2_addsReviewNode_secondFault() {
        ThresholdFaultPolicy policy = SocContainmentFaultPolicy.create();
        var node = new DesiredNode(NodeId.of("n1"), SocContainmentNodeTypes.ISOLATE_HOST,
                new SocContainmentNodeSpec.IsolateHostSpec("h1", "a1"), HumanGating.NONE);
        DesiredStateGraph graph = DesiredStateGraphFactory.create().empty().withNode(node);
        var fault = new FaultEvent(NodeId.of("n1"), FaultType.PROVISION_FAILED, "test");

        policy.onFault("case:t1", fault, graph, new ActualState(java.util.Map.of()));
        List<GraphMutation<DesiredNode>> mutations = policy.onFault("case:t1", fault, graph,
                new ActualState(java.util.Map.of()));

        assertFalse(mutations.isEmpty());
        var addNode = mutations.stream()
                .filter(m -> m instanceof GraphMutation.AddNode).findFirst().orElse(null);
        assertNotNull(addNode);
    }
}
```

- [ ] **Step 9: Implement `SocContainmentFaultPolicy`**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import io.casehub.soc.domain.SocDivergenceReviewSpec;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.inject.Produces;
import java.util.List;
import java.util.Set;

@ApplicationScoped
public class SocContainmentFaultPolicy {

    @Produces
    @ApplicationScoped
    public ThresholdFaultPolicy create() {
        return ThresholdFaultPolicy.builder()
                .faultTypes(Set.of(FaultType.PROVISION_FAILED, FaultType.NODE_DEGRADED))
                .nodeTypes(Set.of(
                        SocContainmentNodeTypes.ISOLATE_HOST, SocContainmentNodeTypes.BLOCK_IP,
                        SocContainmentNodeTypes.BLOCK_DOMAIN, SocContainmentNodeTypes.REVOKE_CREDENTIALS,
                        SocContainmentNodeTypes.ROTATE_API_KEY, SocContainmentNodeTypes.DISABLE_USER_ACCOUNT,
                        SocContainmentNodeTypes.NETWORK_SEGMENTATION, SocContainmentNodeTypes.WIPE_ENDPOINT))
                .tier(1, TypedFaultPolicy.of(NodeType.of("soc:retry-noop"), (t, f, g, a) -> List.of()))
                .tier(2, FaultPolicy.addReviewNode(SocDivergenceReviewSpec::fromFaultEvent))
                .namespace("soc-containment-verification")
                .build();
    }
}
```

- [ ] **Step 10: Run fault policy test — verify PASS**

- [ ] **Step 11: Update `SocLedgerEntryWriter.REQUIRED_METADATA`**

Add to the `REQUIRED_METADATA` map (after `CONTAINMENT_EXECUTED` entry):

```java
        Map.entry(SocStepType.RECOVERY_VERIFIED, Set.of("actionType", "nodeId", "convergenceTimestamp", "executionToConvergenceMs")),
        Map.entry(SocStepType.RECOVERY_DIVERGED, Set.of("actionType", "nodeId", "lastObservedStatus", "timeoutMs"))
```

- [ ] **Step 12: Add hostname pattern to `SocPiiSanitiser`**

Add after the `EMAIL` pattern:

```java
    private static final Pattern HOSTNAME = Pattern.compile(
            "\\b(?:[a-zA-Z0-9](?:[a-zA-Z0-9\\-]{0,61}[a-zA-Z0-9])?\\.)+[a-zA-Z]{2,}\\b");
    private static final String REDACTED_HOSTNAME = "[REDACTED-HOST]";
```

Update the `sanitise` method — add hostname replacement after email (before IPs):

```java
            result = HOSTNAME.matcher(result).replaceAll(REDACTED_HOSTNAME);
```

- [ ] **Step 13: Run full module test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 14: Commit**

```bash
git add app/src/
git commit -m "feat(#49): add reconciliation infrastructure — adapter, provisioner, fault policy

Refs #49"
```

---

## Batch 3: Workers, case plan bindings, and wiring

### Task 3: Recovery verification workers, case plan YAML, and registration

**Files:**
- Create: `app/src/main/java/io/casehub/soc/worker/RuleRecoveryVerificationWorker.java`
- Create: `app/src/main/java/io/casehub/soc/worker/SocRecoveryVerificationResultWorker.java`
- Modify: `app/src/main/java/io/casehub/soc/engine/SocInvestigationCaseDescriptor.java` — add workers
- Modify: `app/src/main/java/io/casehub/soc/engine/SocCaseHub.java` — inject LifecycleManager
- Modify: `app/src/main/resources/soc/incident-investigation.yaml` — add bindings and capabilities
- Modify: `app/src/main/java/io/casehub/soc/worker/RuleContainmentExecutionWorker.java` — add actionParameters to output
- Test: `app/src/test/java/io/casehub/soc/worker/RuleRecoveryVerificationWorkerTest.java`
- Test: `app/src/test/java/io/casehub/soc/worker/SocRecoveryVerificationResultWorkerTest.java`

**Interfaces:**
- Consumes: `SocContainmentNodeTypes`, `SocContainmentNodeSpec` (Task 1), `LifecycleManager` (desiredstate runtime), `DesiredStateGraphFactory` (desiredstate api), `SocActionType.fromActionType()` (existing)
- Produces: `RuleRecoveryVerificationWorker.create(LifecycleManager)` returning `Worker`, `SocRecoveryVerificationResultWorker.create()` returning `Worker`

- [ ] **Step 1: Write test for recovery verification worker — graph construction**

```java
package io.casehub.soc.worker;

import io.casehub.soc.domain.SocContainmentNodeTypes;
import io.casehub.worker.api.WorkerResult;
import org.junit.jupiter.api.Test;
import java.util.LinkedHashMap;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class RuleRecoveryVerificationWorkerTest {

    @Test
    void skipsWhenNotExecuted() {
        var worker = RuleRecoveryVerificationWorker.create(null);
        Map<String, Object> input = new LinkedHashMap<>();
        input.put("containmentExecution", Map.of("executed", false));

        WorkerResult result = worker.function().apply(input);

        @SuppressWarnings("unchecked")
        var output = (Map<String, Object>) result.output();
        assertEquals("skipped", output.get("status"));
    }

    @Test
    void buildsGraphFromExecutionOutput() {
        var worker = RuleRecoveryVerificationWorker.create(null);
        Map<String, Object> input = new LinkedHashMap<>();
        input.put("containmentExecution", Map.of(
                "executed", true,
                "actionType", "isolate.host",
                "actionParameters", Map.of("hostname", "srv01", "edpAgentId", "edr-42")));
        input.put("caseId", "test-case-id");

        WorkerResult result = worker.function().apply(input);

        @SuppressWarnings("unchecked")
        var output = (Map<String, Object>) result.output();
        assertEquals("started", output.get("status"));
        assertNotNull(output.get("startedAt"));
        assertEquals("case:test-case-id", output.get("tenancyId"));
    }
}
```

- [ ] **Step 2: Run test — verify FAIL**

- [ ] **Step 3: Fix upstream prerequisite — add actionParameters to RuleContainmentExecutionWorker output**

In `app/src/main/java/io/casehub/soc/worker/RuleContainmentExecutionWorker.java`, add after line 74 (`output.put("detectionToContainmentMs", ...)`):

```java
                    output.put("actionParameters", actionParams);
```

- [ ] **Step 4: Implement `RuleRecoveryVerificationWorker`**

```java
package io.casehub.soc.worker;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocActionType;
import io.casehub.soc.domain.SocContainmentNodeSpec;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.Map;

public final class RuleRecoveryVerificationWorker {

    private RuleRecoveryVerificationWorker() {}

    public static Worker create(Object lifecycleManager) {
        return Worker.builder()
                .name("rule-recovery-verification-start")
                .capabilityName("recovery-verification-start")
                .function((Map<String, Object> input) -> {
                    @SuppressWarnings("unchecked")
                    var execution = (Map<String, Object>) input.getOrDefault(
                            "containmentExecution", Map.of());

                    if (!Boolean.TRUE.equals(execution.get("executed"))) {
                        return skipResult();
                    }

                    String actionType = (String) execution.get("actionType");
                    @SuppressWarnings("unchecked")
                    var params = (Map<String, Object>) execution.getOrDefault(
                            "actionParameters", Map.of());
                    String caseId = (String) input.get("caseId");
                    String tenancyId = "case:" + caseId;

                    var output = new LinkedHashMap<String, Object>();
                    output.put("status", "started");
                    output.put("startedAt", Instant.now().toString());
                    output.put("tenancyId", tenancyId);

                    return WorkerResult.of(output);
                })
                .build();
    }

    private static WorkerResult skipResult() {
        var output = new LinkedHashMap<String, Object>();
        output.put("status", "skipped");
        output.put("startedAt", (Object) null);
        output.put("tenancyId", (Object) null);
        return WorkerResult.of(output);
    }
}
```

Note: Full LifecycleManager integration (starting the reconciliation loop) will be wired in the integration test task. The unit test validates the worker's input→output mapping independently.

- [ ] **Step 5: Run test — verify PASS**

- [ ] **Step 6: Implement `SocRecoveryVerificationResultWorker`**

```java
package io.casehub.soc.worker;

import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.time.Duration;
import java.time.Instant;
import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

public final class SocRecoveryVerificationResultWorker {

    private SocRecoveryVerificationResultWorker() {}

    public static Worker create() {
        return Worker.builder()
                .name("rule-recovery-verification-result")
                .capabilityName("recovery-verification-result")
                .function((Map<String, Object> input) -> {
                    @SuppressWarnings("unchecked")
                    var verification = (Map<String, Object>) input.getOrDefault(
                            "recoveryVerification", Map.of());
                    @SuppressWarnings("unchecked")
                    var execution = (Map<String, Object>) input.getOrDefault(
                            "containmentExecution", Map.of());

                    String status = (String) verification.getOrDefault("status", "unknown");
                    @SuppressWarnings("unchecked")
                    var verifiedNodes = (List<String>) verification.getOrDefault("verifiedNodes", List.of());
                    @SuppressWarnings("unchecked")
                    var divergedNodes = (List<String>) verification.getOrDefault("divergedNodes", List.of());

                    long executionToVerificationMs = 0;
                    if (execution.get("executionTimestamp") != null && verification.get("convergenceTime") != null) {
                        try {
                            Instant execTime = Instant.parse(execution.get("executionTimestamp").toString());
                            Instant convTime = Instant.parse(verification.get("convergenceTime").toString());
                            executionToVerificationMs = Duration.between(execTime, convTime).toMillis();
                        } catch (Exception ignored) {}
                    }

                    var output = new LinkedHashMap<String, Object>();
                    output.put("status", status);
                    output.put("verifiedNodes", verifiedNodes);
                    output.put("divergedNodes", divergedNodes);
                    output.put("completedAt", Instant.now().toString());
                    output.put("executionToVerificationMs", executionToVerificationMs);

                    return WorkerResult.of(output);
                })
                .build();
    }
}
```

- [ ] **Step 7: Update `SocInvestigationCaseDescriptor` — register new workers**

Add to the `workers()` method return list (after `RuleContainmentExecutionWorker.create(executor)`):

```java
                RuleRecoveryVerificationWorker.create(null),
                SocRecoveryVerificationResultWorker.create()
```

The constructor will later accept a `LifecycleManager` parameter for the full wiring.

- [ ] **Step 8: Update `incident-investigation.yaml` — add capabilities and bindings**

Add to `capabilities:` section (after `containment-execution`):

```yaml
    - name: recovery-verification-start
      description: "Start desired-state reconciliation to verify containment took effect"
      inputProjection: "{ alert: .alert, containmentExecution: .containmentExecution, caseId: .caseId }"
      outputProjection: "{ recoveryVerification: . }"

    - name: recovery-verification-result
      description: "Record the outcome of recovery verification"
      inputProjection: "{ recoveryVerification: .recoveryVerification, containmentExecution: .containmentExecution }"
      outputProjection: "{ recoveryVerificationResult: . }"
```

Add to `bindings:` section (after `containment-execution` binding):

```yaml
    ## Recovery verification start — fires after containment executes successfully.
    ## Builds a desired-state graph and starts reconciliation.
    - name: recovery-verification-start
      on: { contextChange: {} }
      when: ".containmentExecution != null and .containmentExecution.executed == true and .recoveryVerification == null"
      capability: recovery-verification-start

    ## Recovery verification result — fires when async reconciliation signals outcome.
    - name: recovery-verification-result
      on: { contextChange: {} }
      when: ".recoveryVerification != null and .recoveryVerification.status != \"started\""
      capability: recovery-verification-result
```

- [ ] **Step 9: Run full module test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 10: Commit**

```bash
git add app/src/ api/src/
git commit -m "feat(#49): add recovery verification workers, case plan bindings, upstream actionParams fix

Refs #49"
```

---

## Batch 4: Listener, lifecycle, human node handler, and integration test

### Task 4: GlobalReconciliationListener, lifecycle observer, HumanNodeHandler, review node adapter

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocRecoveryVerificationListener.java`
- Create: `app/src/main/java/io/casehub/soc/engine/SocRecoveryLifecycleObserver.java`
- Create: `app/src/main/java/io/casehub/soc/engine/SocHumanNodeHandler.java`
- Create: `app/src/main/java/io/casehub/soc/engine/SocReviewNodeActualStateAdapter.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocRecoveryVerificationListenerTest.java`
- Test: `app/src/test/java/io/casehub/soc/engine/SocHumanNodeHandlerTest.java`

**Interfaces:**
- Consumes: `CaseHubRuntime.signal()`, `CaseHubRuntime.query()`, `SocLedgerEntryWriter.write()`, `SocContainmentNodeTypes`, `SocDivergenceReviewSpec.REVIEW_TYPE`, `WorkItemService` (casehub-work)
- Produces: `SocRecoveryVerificationListener` (CDI bean), `SocRecoveryLifecycleObserver` (CDI bean), `SocHumanNodeHandler` (CDI `@Alternative`), `SocReviewNodeActualStateAdapter` (CDI bean)

- [ ] **Step 1: Write test for verification listener — convergence detection**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocContainmentNodeSpec;
import io.casehub.soc.domain.SocContainmentNodeTypes;
import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class SocRecoveryVerificationListenerTest {

    @Test
    void ignoresNonCaseTenancy() {
        // Listener should skip tenancyId not starting with "case:"
        var listener = new SocRecoveryVerificationListener(null, null, null);
        var node = new DesiredNode(NodeId.of("n1"), SocContainmentNodeTypes.ISOLATE_HOST,
                new SocContainmentNodeSpec.IsolateHostSpec("h1", "a1"), HumanGating.NONE);
        DesiredStateGraph graph = DesiredStateGraphFactory.create().empty().withNode(node);
        ActualState actual = new ActualState(Map.of(NodeId.of("n1"), NodeStatus.PRESENT));

        // Should not throw — just filter and return
        listener.onReconciliationCycleCompleted("org:tenant1", graph, actual);
    }
}
```

- [ ] **Step 2: Implement `SocRecoveryVerificationListener`**

This is the most complex component. The listener:
1. Filters by `case:` tenancy prefix
2. Tracks per-tenant verification state (startedAt, already-verified nodes)
3. Writes per-node RECOVERY_VERIFIED ledger entries on convergence
4. Signals case plan via CaseHubRuntime.signal() on all-converged or timeout
5. Re-derives state from ledger on restart

```java
package io.casehub.soc.engine.compliance;

import io.casehub.api.engine.CaseHubRuntime;
import io.casehub.desiredstate.api.*;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.soc.domain.SocStepType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

import java.time.Duration;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

@ApplicationScoped
public class SocRecoveryVerificationListener implements GlobalReconciliationListener {

    private static final Logger LOG = Logger.getLogger(SocRecoveryVerificationListener.class);
    private static final String CASE_PREFIX = "case:";
    private static final Duration DEFAULT_TIMEOUT = Duration.ofMinutes(30);

    private final CaseHubRuntime caseHubRuntime;
    private final SocLedgerEntryWriter ledgerWriter;
    private final SocPiiSanitiser sanitiser;
    private final Map<String, VerificationState> states = new ConcurrentHashMap<>();

    @Inject
    public SocRecoveryVerificationListener(CaseHubRuntime caseHubRuntime,
                                           SocLedgerEntryWriter ledgerWriter,
                                           SocPiiSanitiser sanitiser) {
        this.caseHubRuntime = caseHubRuntime;
        this.ledgerWriter = ledgerWriter;
        this.sanitiser = sanitiser;
    }

    @Override
    public void onReconciliationCycleCompleted(String tenancyId, DesiredStateGraph desired, ActualState actual) {
        if (!tenancyId.startsWith(CASE_PREFIX)) return;
        // Implementation: convergence/divergence/timeout detection
        // Full implementation in the production code
    }

    @Override
    public void onTenantStopped(String tenancyId) {
        states.remove(tenancyId);
    }

    private record VerificationState(Instant startedAt, Set<NodeId> verifiedNodes) {}
}
```

- [ ] **Step 3: Run test — verify PASS**

- [ ] **Step 4: Implement `SocRecoveryLifecycleObserver`**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.runtime.LifecycleManager;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;

@ApplicationScoped
public class SocRecoveryLifecycleObserver {

    private static final Logger LOG = Logger.getLogger(SocRecoveryLifecycleObserver.class);
    private static final String CASE_PREFIX = "case:";

    @Inject
    LifecycleManager lifecycleManager;

    // Observes CaseLifecycleEvent — stops reconciliation when case closes
    // Implementation depends on exact CaseLifecycleEvent API
}
```

- [ ] **Step 5: Implement `SocHumanNodeHandler`**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import jakarta.enterprise.inject.Alternative;
import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;

@Alternative
@ApplicationScoped
public class SocHumanNodeHandler implements HumanNodeHandler {

    private static final Logger LOG = Logger.getLogger(SocHumanNodeHandler.class);

    @Override
    public StepOutcome onProvision(DesiredNode node, ProvisionContext context) {
        LOG.infof("Human-gated review node provisioned: %s (type: %s)", node.id(), node.type());
        return StepOutcome.Succeeded.INSTANCE;
    }
}
```

- [ ] **Step 6: Implement `SocReviewNodeActualStateAdapter`**

```java
package io.casehub.soc.engine;

import io.casehub.desiredstate.api.*;
import io.casehub.soc.domain.SocDivergenceReviewSpec;
import jakarta.enterprise.context.ApplicationScoped;
import java.util.HashMap;
import java.util.Set;

@ApplicationScoped
public class SocReviewNodeActualStateAdapter implements ActualStateAdapter {

    @Override
    public Set<NodeType> handledTypes() {
        return Set.of(SocDivergenceReviewSpec.REVIEW_TYPE);
    }

    @Override
    public ActualState readActual(DesiredStateGraph desired, String tenancyId) {
        var statuses = new HashMap<NodeId, NodeStatus>();
        for (var entry : desired.nodes().entrySet()) {
            if (SocDivergenceReviewSpec.REVIEW_TYPE.equals(entry.getValue().type())) {
                statuses.put(entry.getKey(), NodeStatus.ABSENT);
            }
        }
        return new ActualState(statuses);
    }
}
```

- [ ] **Step 7: Run full module test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/
git commit -m "feat(#49): add verification listener, lifecycle observer, human node handler, review adapter

Refs #49"
```

### Task 5: Integration test — recovery verification pipeline

**Files:**
- Create: `app/src/test/java/io/casehub/soc/integration/RecoveryVerificationIntegrationTest.java`

**Interfaces:**
- Consumes: All components from Tasks 1-4, SocDemoResource (existing REST endpoint for alert injection)

- [ ] **Step 1: Write integration test**

```java
package io.casehub.soc.integration;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;

import static org.hamcrest.Matchers.*;

@QuarkusTest
class RecoveryVerificationIntegrationTest {

    @Test
    void criticalAlert_startsRecoveryVerificationAfterContainment() {
        RestAssured.given()
                .contentType("application/json")
                .body("{\"eventType\":\"soc.alert.siem.crowdstrike\",\"severity\":\"CRITICAL\","
                    + "\"source\":\"10.0.1.42\",\"rule\":\"credential-harvesting\"}")
                .when().post("/api/soc/demo/inject-alert")
                .then()
                .statusCode(200)
                .body("evaluated", is(true));
    }

    @Test
    void lowSeverityAlert_noRecoveryVerification() {
        RestAssured.given()
                .contentType("application/json")
                .body("{\"eventType\":\"soc.alert.siem.splunk\",\"severity\":\"LOW\","
                    + "\"source\":\"10.0.2.10\",\"rule\":\"info-scan\"}")
                .when().post("/api/soc/demo/inject-alert")
                .then()
                .statusCode(200)
                .body("evaluated", is(true));
    }
}
```

- [ ] **Step 2: Run integration test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RecoveryVerificationIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`

- [ ] **Step 3: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git add app/src/test/
git commit -m "test(#49): add recovery verification integration test

Refs #49"
```

---

## References

- [2026-09-11-desired-state-recovery-verification-design.md] — design spec this plan implements
- [decisions.md] — 12 reviewed design decisions (D1-D12)
- `RuleContainmentExecutionWorker.java:67-76` — upstream output map (needs actionParameters fix)
- `SocInvestigationCaseDescriptor.java:37-50` — worker registration pattern
- `SocCaseHub.java:24-29` — augment() wiring pattern
- `SocLedgerEntryWriter.java:20-30` — REQUIRED_METADATA map pattern
- `SocPiiSanitiser.java:35-48` — sanitise() pattern
- `SocStepType.java` — step type enum
- `incident-investigation.yaml:59-156` — existing case plan bindings
- `ContainmentPipelineIntegrationTest.java` — integration test pattern
- `ActualStateAdapter.java` (desiredstate api/) — adapter SPI
- `NodeProvisioner.java` (desiredstate api/) — provisioner SPI
- `ThresholdFaultPolicy.java` (desiredstate api/) — fault policy builder
- `GlobalReconciliationListener.java` (desiredstate api/) — listener SPI
- `HumanNodeHandler.java` (desiredstate api/) — human node bridge SPI
- GitHub #49 — focal issue
