# YAML Helpdesk Apps — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create epic before starting
**Issue group:** Spans casehub-engine, casehub-eidos, casehub-pages, casehubio/examples

**Goal:** A complete helpdesk application defined entirely in YAML — case lifecycle, eidos agents, LLM/simulator classification, dashboards — zero Java. Side by side with the existing Java helpdesk.

**Architecture:** The case engine auto-discovers case YAML and uses a JSON collection SPI adapter for persistence. Workers call eidos agents via `call: agent` (wired to LLM or simulator). Push topics serve as the event bus. Pages renders the UI from page YAML. Playbooks serve as both teaching walkthroughs and headless integration tests.

**Tech Stack:** Quarkus (CDI, REST), casehub-engine (case definitions, bindings, workers), casehub-eidos (agent descriptors, registry), casehub-pages (data-json, push, playbooks, visualiser), Playwright (headless testing), Lit/TypeScript (decorators)

## Global Constraints

- Java 26, Quarkus 3.x, Maven
- All new YAML schemas must follow the DSL Style Guide (§YAML/Java Parity Principle)
- Zero Java in the helpdesk-apps module — runtime deps only
- Simulator mode must be the default; LLM mode opt-in via config property
- ARIA selectors for all playbook UI steps
- Every batch must leave all repos in a compilable, tests-passing state

---

## Batch 1: Case Engine Auto-Discovery

**Repo:** casehub-engine
**After this batch:** Case definitions on the classpath at `casehub/*.yaml` are automatically registered as CaseHub beans — no Java class needed.

### Task 1: Auto-discovery CDI extension

**Files:**
- Create: `casehub-engine/flow/src/main/java/io/casehub/engine/flow/discovery/YamlCaseHubDiscovery.java`
- Create: `casehub-engine/flow/src/test/java/io/casehub/engine/flow/discovery/YamlCaseHubDiscoveryTest.java`
- Modify: `casehub-engine/flow/src/main/resources/META-INF/services/` (CDI extension registration)

**Interfaces:**
- Consumes: `YamlCaseHub` constructor (existing — takes resource path string)
- Consumes: `FlowWorkerFunctionProvider` (existing — resolves `do:` blocks)
- Produces: CDI beans of type `CaseHub` for each discovered YAML file

- [ ] **Step 1: Write failing test — verify auto-discovery finds a classpath YAML**

Create a test case YAML at `src/test/resources/casehub/test-auto-discovery.yaml` with a minimal case definition (one capability, one worker with a `do:` block, one binding). Write a `@QuarkusTest` that injects `Instance<CaseHub>` and asserts a bean was created for the test YAML.

```java
@QuarkusTest
class YamlCaseHubDiscoveryTest {
    @Inject
    Instance<CaseHub> caseHubs;

    @Test
    void autoDiscoveredCaseHubExists() {
        var hub = caseHubs.stream()
            .filter(h -> h.name().equals("test-auto-discovery"))
            .findFirst();
        assertThat(hub).isPresent();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f casehub-engine/flow/pom.xml test -Dtest=YamlCaseHubDiscoveryTest -s .mvn/slot-settings.xml`
Expected: FAIL — no auto-discovered bean

- [ ] **Step 3: Implement the CDI extension**

Create `YamlCaseHubDiscovery` as a `@Startup` `@ApplicationScoped` bean that:
1. Scans classpath for `casehub/*.yaml` resources
2. Filters out any already registered by explicit Java CaseHub classes
3. For each unregistered YAML, creates and initializes a `YamlCaseHub` instance
4. Registers it in the `CaseHubRegistry` (or equivalent CDI producer)

Use `ide_insert_member` for new methods. Reference `ClasspathYamlDescriptorRegistrar` in eidos for the classpath scanning pattern.

- [ ] **Step 4: Run test to verify it passes**

Run: `mvn -f casehub-engine/flow/pom.xml test -Dtest=YamlCaseHubDiscoveryTest -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 5: Verify existing explicit CaseHubs still work**

Run the full engine test suite to confirm no regression — explicit Java CaseHub classes (like `HelpDeskCaseHub`) must still take precedence over auto-discovery.

Run: `mvn -f casehub-engine/flow/pom.xml test -s .mvn/slot-settings.xml`
Expected: All existing tests PASS

- [ ] **Step 6: Commit**

```bash
git add casehub-engine/flow/src/
git commit -m "feat(engine): auto-discover case YAML on classpath

Scans casehub/*.yaml on classpath at startup and registers CaseHub
beans for each. Explicit Java CaseHub classes take precedence.
Enables zero-Java case applications.

Refs #<N>"
```

---

## Batch 2: JSON Collection SPI Adapter

**Repo:** casehub-engine + casehub-pages (data-json)
**After this batch:** The case engine can use JSON collections (SQLite-backed) as its persistence layer — standard SPI, uniform data access.

### Task 2: JSON collection persistence adapter

**Files:**
- Create: `casehub-engine/persistence-json/pom.xml` (new module)
- Create: `casehub-engine/persistence-json/src/main/java/io/casehub/engine/persistence/json/JsonCollectionCaseInstanceCache.java`
- Create: `casehub-engine/persistence-json/src/test/java/io/casehub/engine/persistence/json/JsonCollectionCaseInstanceCacheTest.java`
- Modify: `casehub-engine/pom.xml` (add module)

**Interfaces:**
- Consumes: `CaseInstanceCache` SPI (existing — the engine's persistence interface)
- Consumes: `JsonCollectionStore` APIs (existing in data-json — CRUD, query, optimistic concurrency)
- Produces: `CaseInstanceCache` implementation backed by JSON collections

- [ ] **Step 1: Explore the CaseInstanceCache SPI**

Use `ide_find_class` to locate `CaseInstanceCache` and understand its interface. Use `ide_find_implementations` to see existing implementations (in-memory, JPA). Document the methods that need implementation.

- [ ] **Step 2: Create the new Maven module**

Create `casehub-engine/persistence-json/pom.xml` with dependencies on `casehub-engine-api` and `casehub-pages-data-json`.

- [ ] **Step 3: Write failing test for CRUD operations**

```java
@QuarkusTest
class JsonCollectionCaseInstanceCacheTest {
    @Inject
    JsonCollectionCaseInstanceCache cache;

    @Test
    void createAndRetrieve() {
        var instance = CaseInstance.builder()
            .name("helpdesk-ticket")
            .context(Map.of("subject", "Laptop won't boot", "status", "OPEN"))
            .build();
        var id = cache.create(instance);
        var retrieved = cache.get(id);
        assertThat(retrieved.context().get("subject")).isEqualTo("Laptop won't boot");
    }

    @Test
    void updateWithOptimisticConcurrency() {
        // Create, update, verify version increments
    }
}
```

- [ ] **Step 4: Implement JsonCollectionCaseInstanceCache**

Implement `CaseInstanceCache` by delegating to `JsonCollectionStore`:
- `create()` → `POST /api/apps/{appId}/{collection}` or direct store call
- `get()` → query by document ID
- `update()` → PATCH with `_version` for optimistic concurrency
- `query()` → translate case query to `DataSetLookup` filter

- [ ] **Step 5: Run tests, verify pass**

- [ ] **Step 6: Integration test with case engine**

Write a test that creates a case using the auto-discovered YAML (from Batch 1) backed by JSON collections. Verify the full create → query → update cycle works through the engine.

- [ ] **Step 7: Commit**

```bash
git add casehub-engine/persistence-json/
git commit -m "feat(engine): JSON collection SPI adapter for case persistence

Standard CaseInstanceCache implementation backed by JsonCollectionStore.
Uniform data access — case instances stored as JSON documents with
optimistic concurrency via _version column.

Refs #<N>"
```

---

## Batch 3: Eidos Agent Dispatch from Case YAML

**Repo:** casehub-eidos + casehub-engine
**After this batch:** Workers in case YAML can reference eidos agents by slot and invoke them via `call: agent`. Agent-task bindings use capability matching via `AgentRegistry`.

### Task 3: `call: agent` worker function provider

**Files:**
- Create: `casehub-engine/flow/src/main/java/io/casehub/engine/flow/worker/AgentWorkerFunctionProvider.java`
- Create: `casehub-engine/flow/src/test/java/io/casehub/engine/flow/worker/AgentWorkerFunctionProviderTest.java`
- Modify: worker provider SPI registration

**Interfaces:**
- Consumes: `WorkerFunctionProvider` SPI (existing — `FlowWorkerFunctionProvider` pattern)
- Consumes: `AgentRegistry` (existing in eidos — resolve agent by slot)
- Consumes: `SystemPromptRenderer` (existing in eidos — render descriptor into prompt)
- Consumes: Agent-config provider resolution (existing — LLM or simulator wiring)
- Produces: `WorkerFunction` that calls eidos agent with prompt from `do:` block

- [ ] **Step 1: Explore WorkerFunctionProvider SPI**

Use `ide_find_class` for `WorkerFunctionProvider` and `ide_find_implementations` to see the existing providers (`FlowWorkerFunctionProvider`, `McpWorkerFunctionProvider`). Understand the trigger mechanism (how does the provider claim a worker definition).

- [ ] **Step 2: Write failing test — worker with `call: agent` resolves to eidos agent**

Create a test case YAML with a worker that has `agent: test-agent` and a `do:` block with `call: agent`. Register a test eidos descriptor. Assert the worker resolves and invokes the agent.

- [ ] **Step 3: Implement AgentWorkerFunctionProvider**

Triggered by presence of `agent:` field on worker definition. Resolves the eidos descriptor via `AgentRegistry.find(slot)`, renders system prompt via `SystemPromptRenderer`, invokes via agent-config provider (LLM or simulator). Returns structured output matching the `output-schema` in the `do:` block.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

### Task 4: Agent-task binding resolver

**Files:**
- Modify: Binding resolution code in casehub-engine (the class that handles `human-task:` bindings)
- Create: test for `agent-task:` binding type

**Interfaces:**
- Consumes: `AgentRegistry.find()` with capability matching (existing in eidos)
- Consumes: `CapabilityHealth.probe()` (existing in eidos — agent availability)
- Produces: `agent-task:` binding type that routes to eidos agents by capability

- [ ] **Step 1: Explore existing human-task binding resolution**

Use `ide_search_text` for `human-task` in the engine to find where bindings are resolved. Understand how candidate groups are currently determined.

- [ ] **Step 2: Write failing test — agent-task resolves via capability matching**

```java
@Test
void agentTaskResolvesToCapableAgent() {
    // Case YAML has: agent-task: { capability: "hardware-support" }
    // Eidos has: hw-specialist with capability: hardware-support
    // Assert: work item created with hw-specialist as candidate
}
```

- [ ] **Step 3: Implement agent-task binding type**

Add `agent-task:` as a new binding action type. At binding fire time:
1. Evaluate the capability expression (JQ)
2. Call `AgentRegistry.find()` with the capability
3. Probe availability via `CapabilityHealth`
4. Create work item with matched agents as candidates

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(engine): call: agent and agent-task: bindings for eidos dispatch

Workers can reference eidos agents by slot and invoke via call: agent.
Agent-task bindings use capability matching via AgentRegistry instead
of hardcoded candidate groups.

Refs #<N>"
```

---

## Batch 4: Push Topic Publish Endpoint

**Repo:** casehub-pages-push
**After this batch:** Workers' `do:` blocks can publish events to push topics via REST.

### Task 5: REST publish endpoint

**Files:**
- Create: `casehub-pages/backend/push-runtime/src/main/java/io/casehub/pages/push/PushPublishResource.java`
- Create: `casehub-pages/backend/push-runtime/src/test/java/io/casehub/pages/push/PushPublishResourceTest.java`

**Interfaces:**
- Consumes: Push broadcast infrastructure (existing — `TopicBroadcaster` or equivalent)
- Produces: `POST /api/push/publish/{topic}` REST endpoint

- [ ] **Step 1: Explore existing push broadcast infrastructure**

Use `ide_find_class` for the topic broadcaster/publisher in push-runtime. Understand how `TicketPushObserver` currently publishes to topics.

- [ ] **Step 2: Write failing test**

```java
@QuarkusTest
class PushPublishResourceTest {
    @Test
    void publishToTopicBroadcasts() {
        // POST /api/push/publish/test:topic with JSON body
        // Assert: WebSocket subscriber on that topic receives the message
        given()
            .contentType(ContentType.JSON)
            .body(Map.of("key", "value"))
            .post("/api/push/publish/test:topic")
            .then()
            .statusCode(200);
        // Verify via WebSocket test client
    }
}
```

- [ ] **Step 3: Implement PushPublishResource**

Thin REST wrapper: accept JSON payload, broadcast to topic subscribers via existing infrastructure. Add authentication consistent with existing push endpoints.

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

```bash
git commit -m "feat(push): REST endpoint for publishing to push topics

POST /api/push/publish/{topic} broadcasts JSON payload to all
subscribers. Enables YAML workers to publish events via call: http.

Refs #<N>"
```

---

## Batch 5: YAML Helpdesk Example — Data + Case Lifecycle

**Repo:** casehubio/examples
**After this batch:** The `helpdesk-apps/` module exists with app.yaml, case YAML, eidos descriptors, agent manifest, seed data. The case lifecycle works end-to-end (create → classify → assign → resolve → notify) in simulator mode.

### Task 6: Create helpdesk-apps module with YAML files

**Files:**
- Create: `helpdesk-apps/pom.xml`
- Create: `helpdesk-apps/src/main/resources/org/apps/helpdesk/app.yaml`
- Create: `helpdesk-apps/src/main/resources/org/apps/helpdesk/data/tickets.json`
- Create: `helpdesk-apps/src/main/resources/org/apps/helpdesk/data/agents.json`
- Create: `helpdesk-apps/src/main/resources/casehub/helpdesk-ticket.yaml`
- Create: `helpdesk-apps/src/main/resources/META-INF/eidos/descriptors.yaml`
- Create: `helpdesk-apps/src/main/resources/META-INF/eidos/agent-manifest.yaml`
- Create: `helpdesk-apps/src/main/resources/application.properties`

**Interfaces:**
- Consumes: Auto-discovery (Batch 1), JSON collection SPI (Batch 2), eidos dispatch (Batch 3), push publish (Batch 4)
- Produces: Running helpdesk app with full case lifecycle

- [ ] **Step 1: Create pom.xml with runtime dependencies**

Dependencies: `casehub-engine`, `casehub-engine-flow`, `casehub-engine-persistence-json`, `casehub-eidos`, `casehub-pages-data-json`, `casehub-pages-push-runtime`, `casehub-pages-playbook-runtime`. No Java source directories.

- [ ] **Step 2: Create app.yaml**

Collections: tickets (8 fields), agents (4 fields). Seed data references. Access control. Use the schema from the spec verbatim.

- [ ] **Step 3: Create seed data JSON files**

Copy ticket and agent seed data from the Pages helpdesk (12 tickets, 5 agents). Adjust IDs to match the new schema.

- [ ] **Step 4: Create helpdesk-ticket.yaml case definition**

Full case definition with capabilities, workers with `do:` blocks (`call: agent` for classifier, `call: http` for notifier), bindings (triage → assign → notify), and goals. Use the YAML from the spec.

- [ ] **Step 5: Create eidos descriptors.yaml**

Agent descriptors for classifier-agent, hw-specialist, sw-specialist, access-specialist. Capabilities, disposition, vocabulary terms. Use the YAML from the spec.

- [ ] **Step 6: Create agent-manifest.yaml**

Dual-wired: `classifier-llm` (Anthropic) and `classifier-sim` (simulator with keyword rules). Default active: `classifier-sim`.

- [ ] **Step 7: Create application.properties**

Quarkus config: port 8190 (different from helpdesk's 8090), H2 in-memory, Flyway, CORS, simulator as default provider.

- [ ] **Step 8: Build and verify startup**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f helpdesk-apps/pom.xml install -Dmaven.test.skip=true -s .mvn/slot-settings.xml
```

- [ ] **Step 9: Commit**

```bash
git add helpdesk-apps/
git commit -m "feat(helpdesk-apps): YAML-only helpdesk with eidos + simulator

Zero Java. Case lifecycle (classify → assign → resolve → notify),
eidos agent descriptors, LLM/simulator dual-wiring. Auto-discovered
case definition, JSON collection persistence.

Refs #<N>"
```

### Task 7: End-to-end lifecycle validation test

**Files:**
- Create: `helpdesk-apps/src/test/java/io/casehub/examples/helpdesk/apps/LifecycleE2ETest.java`

**Note:** This is the ONE test file — a Java test that validates the YAML-only app works. The app itself has zero Java; the test proves it.

**Interfaces:**
- Consumes: All Batch 1-5 platform capabilities
- Produces: Green test proving the full lifecycle

- [ ] **Step 1: Write the E2E test**

```java
@QuarkusTest
class LifecycleE2ETest {
    @Test
    void fullLifecycle_simulatorMode() {
        // 1. Create ticket via REST
        var ticketId = given()
            .contentType(ContentType.JSON)
            .body(Map.of("subject", "Laptop won't boot", "status", "OPEN"))
            .post("/api/apps/helpdesk/tickets")
            .then().statusCode(201)
            .extract().path("id");

        // 2. Wait for classification binding to fire (simulator)
        await().atMost(5, SECONDS).untilAsserted(() -> {
            var ticket = get("/api/apps/helpdesk/tickets/" + ticketId)
                .then().extract().as(Map.class);
            assertThat(ticket.get("category")).isEqualTo("HARDWARE");
            assertThat(ticket.get("status")).isEqualTo("TRIAGED");
        });

        // 3. Verify work item created for hw-specialist
        await().atMost(5, SECONDS).untilAsserted(() -> {
            var items = get("/api/work/items/list-all")
                .then().extract().as(List.class);
            assertThat(items).isNotEmpty();
        });

        // 4. Claim and resolve work item
        // ... claim via REST, resolve with resolution text

        // 5. Verify notification sent
        await().atMost(5, SECONDS).untilAsserted(() -> {
            var ticket = get("/api/apps/helpdesk/tickets/" + ticketId)
                .then().extract().as(Map.class);
            assertThat(ticket.get("status")).isEqualTo("RESOLVED");
            assertThat(ticket.get("notified")).isEqualTo(true);
        });
    }
}
```

- [ ] **Step 2: Run test, verify pass**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f helpdesk-apps/pom.xml test -Dtest=LifecycleE2ETest -s .mvn/slot-settings.xml
```

- [ ] **Step 3: Commit**

---

## Batch 6: Helpdesk UI — Page YAML + Playbook

**Repo:** casehubio/examples
**After this batch:** The helpdesk-apps module has a full UI (dashboard, queue, forms) and an interactive demo playbook with business logic assertions.

### Task 8: Page YAML

**Files:**
- Create: `helpdesk-apps/src/main/resources/org/apps/helpdesk/helpdesk.page.yaml`

**Interfaces:**
- Consumes: `mutableRestSource` (existing in Pages — CRUD bindings)
- Consumes: `collectionBindings()` (existing in Pages — auto-wired endpoints)
- Produces: 6 pages: index, dashboard, ticket queue, ticket detail, new ticket, agents

- [ ] **Step 1: Copy and adapt the page YAML from Pages repo**

Start from `/Users/mdproctor/claude/casehub/pages/examples/server/src/main/resources/org/apps/helpdesk/helpdesk.page.yaml`. Adapt endpoint URLs to match the helpdesk-apps module's routing.

- [ ] **Step 2: Start dev mode, verify UI renders**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f helpdesk-apps/pom.xml quarkus:dev -Dquarkus.http.port=8190 -s .mvn/slot-settings.xml
```

Navigate to `http://127.0.0.1:8190/` — verify dashboard, forms, tables render with seed data.

- [ ] **Step 3: Commit**

### Task 9: Demo playbook with assertions

**Files:**
- Create: `helpdesk-apps/src/main/resources/playbooks/helpdesk-demo.yaml`

**Interfaces:**
- Consumes: Playbook infrastructure (existing — step execution, verification)
- Produces: Dual-purpose playbook (teaching + testing) with ARIA selectors and verify: assertions

- [ ] **Step 1: Write the playbook YAML**

Full lifecycle demo with ARIA selectors and `verify:` assertion steps. Use the YAML from the spec. Steps: bootstrap → submit ticket → verify classification → verify work item → claim/resolve → verify notification → verify case complete.

- [ ] **Step 2: Run in browser, verify teaching mode works**

Start dev mode, navigate to the app, trigger the playbook. Watch each step execute and verify checkmarks appear on `verify:` steps.

- [ ] **Step 3: Commit**

---

## Batch 7: TS Decorator Framework

**Repo:** casehub-pages (new module)
**After this batch:** TypeScript decorators (`@Observes`, `@Publish`, `@ApplicationScoped`) wired to push topics. Same paradigm as YAML bindings, code-first syntax for web developers.

### Task 10: Core decorator implementation

**Files:**
- Create: `casehub-pages/packages/event-decorators/src/observes.ts`
- Create: `casehub-pages/packages/event-decorators/src/publish.ts`
- Create: `casehub-pages/packages/event-decorators/src/scoped.ts`
- Create: `casehub-pages/packages/event-decorators/src/container.ts`
- Create: `casehub-pages/packages/event-decorators/src/index.ts`
- Create: `casehub-pages/packages/event-decorators/package.json`
- Create: `casehub-pages/packages/event-decorators/tsconfig.json`
- Create: `casehub-pages/packages/event-decorators/tests/decorators.test.ts`

**Interfaces:**
- Consumes: Push WebSocket infrastructure (existing — `EventStreamController` or similar)
- Produces: `@Observes(topic)`, `@Publish(topic)`, `@ApplicationScoped` decorators

- [ ] **Step 1: Write failing test**

```typescript
import { Observes, Publish, ApplicationScoped, EventContainer } from '../src';

@ApplicationScoped
class TestService {
  received: string[] = [];

  @Observes('test:events')
  onEvent(event: { message: string }) {
    this.received.push(event.message);
  }

  @Publish('test:results')
  processEvent(input: string): { result: string } {
    return { result: input.toUpperCase() };
  }
}

test('Observes receives topic messages', async () => {
  const container = new EventContainer();
  const service = container.get(TestService);
  // Simulate topic message
  container.dispatch('test:events', { message: 'hello' });
  expect(service.received).toContain('hello');
});

test('Publish sends return value to topic', async () => {
  const container = new EventContainer();
  const service = container.get(TestService);
  const published: any[] = [];
  container.subscribe('test:results', (msg) => published.push(msg));
  service.processEvent('hello');
  expect(published[0]).toEqual({ result: 'HELLO' });
});
```

- [ ] **Step 2: Implement decorators**

`@Observes(topic)`: method decorator. Registers the method as a subscriber to the given push topic. The `EventContainer` wires the subscription at startup.

`@Publish(topic)`: method decorator. Wraps the method — after it returns, publishes the return value to the given push topic.

`@ApplicationScoped`: class decorator. Marks the class as a singleton in the `EventContainer`.

`EventContainer`: lightweight DI container. Scans decorated classes, creates singletons, wires WebSocket subscriptions to push topics.

- [ ] **Step 3: Run tests, verify pass**

- [ ] **Step 4: Wire to WebSocket push infrastructure**

Connect `EventContainer` subscriptions to the actual push WebSocket transport. When `@Observes('topic')` is registered, subscribe to the WebSocket topic. When `@Publish('topic')` fires, send via WebSocket.

- [ ] **Step 5: Commit**

### Task 11: Helpdesk TS decorator example

**Files:**
- Create: `helpdesk-apps/src/main/webui/ts-example/helpdesk-reactions.ts`

**Interfaces:**
- Consumes: Decorator framework (Task 10)
- Produces: Small example showing YAML and TS expressing the same behaviour

- [ ] **Step 1: Write the TS equivalent of the YAML bindings**

```typescript
import { Observes, Publish, ApplicationScoped } from '@casehub/event-decorators';

@ApplicationScoped
export class HelpdeskReactions {
  @Observes('helpdesk:ticket-triaged')
  async onTriaged(event: { ticketId: string; category: string }) {
    console.log(`Ticket ${event.ticketId} triaged as ${event.category}`);
  }

  @Observes('helpdesk:ticket-resolved')
  @Publish('helpdesk:notifications')
  async onResolved(event: { ticketId: string; customerRef: string; resolution: string }) {
    return {
      to: event.customerRef,
      message: `Your ticket has been resolved: ${event.resolution}`
    };
  }
}
```

- [ ] **Step 2: Document the YAML ↔ TS equivalence**

Add a comment block showing the YAML binding and its TS decorator equivalent side by side.

- [ ] **Step 3: Commit**

---

## Batch 8: Three-Way Viewer

**Repo:** casehub-pages
**After this batch:** All YAML artifacts viewable in YAML, visual, or split mode with linked navigation.

### Task 12: Three-way viewer component

**Files:**
- Create: `casehub-pages/packages/ui/src/viewer/three-way-viewer.ts`
- Create: `casehub-pages/packages/ui/src/viewer/yaml-panel.ts`
- Create: `casehub-pages/packages/ui/src/viewer/visual-panel.ts`
- Create: `casehub-pages/packages/ui/src/viewer/viewer-controller.ts`
- Modify: existing playbook viewer component (add split mode)

**Interfaces:**
- Consumes: Existing YAML viewer, existing graphical playbook viewer
- Produces: `<three-way-viewer>` Lit component with mode toggle and linked navigation

- [ ] **Step 1: Explore existing viewer components**

Use `ide_search_text` for the YAML viewer and playbook viewer components in the Pages UI packages.

- [ ] **Step 2: Implement viewer controller**

State machine: YAML | VISUAL | SPLIT. Tracks cursor position for linked navigation.

- [ ] **Step 3: Implement three-way-viewer component**

Lit component that:
- Renders mode toggle buttons (YAML / Visual / Split)
- In YAML mode: shows the YAML text panel
- In Visual mode: shows the graphical flow panel
- In Split mode: shows both side by side
- In Split mode: clicking a visual node scrolls the YAML panel to the corresponding block; clicking a YAML block highlights the visual node

- [ ] **Step 4: Wire into helpdesk page YAML**

Add the three-way viewer to the helpdesk page YAML as a component for viewing case definitions and playbooks.

- [ ] **Step 5: Manual testing in browser**

Start helpdesk-apps dev mode, verify all three modes work. Test linked navigation in split mode.

- [ ] **Step 6: Commit**

---

## Batch 9: Headless Playbook Testing

**Repo:** casehubio/examples + casehub-pages
**After this batch:** Playbooks run as headless Playwright integration tests via CI. ARIA selectors, verify: steps as assertions.

### Task 13: Playwright headless runner

**Files:**
- Create: `casehub-pages/backend/playbook-runtime/src/main/java/io/casehub/pages/playbook/runner/HeadlessPlaybookRunner.java`
- Create: `casehub-pages/backend/playbook-runtime/src/test/java/io/casehub/pages/playbook/runner/HeadlessPlaybookRunnerTest.java`

**Interfaces:**
- Consumes: Playbook YAML parser (existing), `verify:` step type (new — from D9)
- Consumes: Playwright Java API (`com.microsoft.playwright`)
- Produces: Runner that executes playbook steps in headless Chromium, treating `verify:` steps as assertions

- [ ] **Step 1: Add Playwright dependency**

Add `com.microsoft.playwright:playwright` to pom.xml test scope.

- [ ] **Step 2: Write failing test**

Create a minimal playbook with one `fill:` step (ARIA selector) and one `verify:` step. Assert the runner executes both and reports pass/fail.

- [ ] **Step 3: Implement HeadlessPlaybookRunner**

Translate playbook step types to Playwright API calls:
- `fill: { role: textbox, name: "X" }` → `page.getByRole("textbox", { name: "X" }).fill(value)`
- `click: { role: button, name: "X" }` → `page.getByRole("button", { name: "X" }).click()`
- `select: { role: combobox, name: "X" }` → `page.getByRole("combobox", { name: "X" }).selectOption(value)`
- `verify:` → REST call to endpoint, assert expected values, with retry/poll for eventual consistency

- [ ] **Step 4: Run tests, verify pass**

- [ ] **Step 5: Commit**

### Task 14: Helpdesk playbook as integration test

**Files:**
- Create: `helpdesk-apps/src/test/java/io/casehub/examples/helpdesk/apps/PlaybookIntegrationTest.java`

**Interfaces:**
- Consumes: `HeadlessPlaybookRunner` (Task 13), helpdesk-demo.yaml (Task 9)
- Produces: CI-runnable integration test using the demo playbook

- [ ] **Step 1: Write the integration test**

```java
@QuarkusTest
class PlaybookIntegrationTest {
    @Inject
    HeadlessPlaybookRunner runner;

    @Test
    void helpdeskDemoPlaybookPasses() {
        var result = runner.run("playbooks/helpdesk-demo.yaml",
            PlaybookRunnerConfig.builder()
                .baseUrl("http://127.0.0.1:8190")
                .headless(true)
                .timeout(Duration.ofSeconds(30))
                .build());
        assertThat(result.passed()).isTrue();
        assertThat(result.failedSteps()).isEmpty();
    }
}
```

- [ ] **Step 2: Run test against helpdesk-apps in simulator mode**

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f helpdesk-apps/pom.xml test -Dtest=PlaybookIntegrationTest -s .mvn/slot-settings.xml
```

- [ ] **Step 3: Commit**

---

## Batch 10: Polish — Documentation + Visualiser

**Repo:** casehubio/examples + casehub-pages
**After this batch:** Side-by-side documentation, visualiser enhancements for case bindings, and the generate → review → test tutorial.

### Task 15: Side-by-side documentation

**Files:**
- Create: `helpdesk-apps/README.md`

- [ ] **Step 1: Write comparison document**

Markdown document showing the Java vs YAML side-by-side comparison table from the spec. Include: what each file does, how to run each version, how to switch between LLM and simulator modes.

- [ ] **Step 2: Commit**

### Task 16: Visualiser — case binding rendering

**Files:**
- Modify: Playbook visualiser in casehub-pages (extend step model for case bindings)

- [ ] **Step 1: Explore visualiser step model**

Use `ide_find_class` for the playbook visualiser's step/node types. Understand how playbook steps are currently rendered as visual nodes.

- [ ] **Step 2: Add case binding node type**

Extend the visualiser to render:
- Bindings as conditional nodes (trigger condition → worker execution → outcome)
- Worker `do:` blocks as sub-steps
- State transitions as timeline

- [ ] **Step 3: Manual testing — verify case lifecycle renders in visualiser**

- [ ] **Step 4: Commit**

---

## References

- [2026-11-19-yaml-helpdesk-apps-design.md] — design spec this plan implements
- [helpdesk/src/main/resources/casehub/helpdesk-ticket.yaml] — existing case definition
- [helpdesk/src/main/java/.../engine/HelpDeskCaseHub.java] — existing CaseHub (to be replaced by auto-discovery)
- [casehub-pages/examples/server/src/main/resources/org/apps/helpdesk/app.yaml] — existing Pages helpdesk app
- [casehub-pages/examples/server/src/main/resources/org/apps/helpdesk/helpdesk.page.yaml] — existing Pages helpdesk UI
- [casehub-engine/schema/src/main/resources/examples/document-processing.yaml] — existing `do:` block example
- [casehub-eidos eidos-examples] — existing YAML-defined agent descriptors
