# YAML Helpdesk Apps — Design Spec

**Date:** 2026-11-19
**Status:** draft
**Scope:** casehubio/examples, casehub-engine, casehub-pages

## Vision

A complete helpdesk application — case lifecycle, reactive automation, dashboards, forms — defined entirely in YAML. Zero Java. Zero compilation. The same behaviour as the existing 20-class Java helpdesk, expressed in ~5 declarative files.

Two motivations drive this:

1. **LLM-generated apps with visual review.** An LLM generates YAML. The playbook visualiser renders the full lifecycle as an interactive flow. A human reviewer sees the behaviour, not the syntax. This scales where reading code or YAML does not.

2. **Web developer onboarding.** TS decorators (`@Observes`, `@Publish`) provide a familiar code-first authoring mode that compiles to the same runtime. YAML and TS are peer syntaxes — same paradigm, different audiences.

The existing Java helpdesk remains alongside as a comparison. Same behaviour, two authoring models, side by side.

---

## Architecture

### The paradigm

CDI gives Java apps a clean event-driven model: `Event.fire()` and `@Observes`, scoped lifecycle, loose coupling. This paradigm translates directly to both YAML and TypeScript:

```yaml
# YAML: case binding = @Observes + method body
bindings:
  - name: assign-specialist
    trigger:
      on: context-change
      when: "${ .status == \"TRIAGED\" and .resolution == null }"
    do:
      - assign:
          call: http
          with:
            endpoint: /api/work/items/create
            body:
              ticket: "${ . }"
```

```typescript
// TS: decorator = @Observes, method = handler
@Observes('helpdesk:ticket-triaged')
async assignSpecialist(event: TicketEvent) {
  await fetch('/api/work/items/create', { method: 'POST', body: event.ticket });
}
```

Same runtime underneath: bindings + workers + push topics. YAML for declarative apps and LLM generation. TS decorators for web developers who prefer code.

### Two authoring modes, one runtime

| Audience | Authoring mode | Review surface |
|----------|---------------|----------------|
| LLM-generated apps / low-code | YAML | Playbook visualiser |
| Web developers | TS decorators | Code + IDE tooling |

Both compile to the same execution model:
- **Bindings** fire on state changes (case context mutations, topic events)
- **Workers** execute `do:` blocks (Serverless Workflow steps: `call: http`, `switch:`, expressions)
- **Push topics** serve as the event bus (CDI equivalent: pub/sub over WebSocket)

### Data layer

The case engine's data persistence uses a standard SPI. JSON collections are one implementation — same interface, different adapter. The case engine doesn't know or care whether it's backed by in-memory maps, JDBC, or JSON collections in SQLite.

```
┌─────────────────┐     ┌──────────────┐     ┌───────────────────┐
│  Case Engine    │────▶│ DataProvider  │────▶│ JsonCollection-   │
│  (bindings,     │     │ SPI          │     │ Store (SQLite)    │
│   workers,      │     └──────────────┘     └───────────────────┘
│   goals)        │            ▲
└─────────────────┘            │
                        ┌──────────────┐
                        │ In-Memory    │  ← existing adapter
                        │ (current)    │
                        └──────────────┘
```

### Event flow

```
  Ticket created ──▶ Case engine evaluates bindings
                         │
                    status == OPEN, category == null
                         │
                    ▼ classify-ticket binding fires
                         │
                    Worker do: block executes
                    (switch: JQ keyword match)
                         │
                    Context updated: category=HARDWARE, status=TRIAGED
                         │
                    ▼ assign-specialist binding fires
                         │
                    Worker do: block executes
                    (call: http → create work item)
                         │
                    Publishes to push topic: helpdesk:work-items
                         │
                    ▼ Human claims + resolves work item
                         │
                    Context updated: status=RESOLVED
                         │
                    ▼ notify-resolution binding fires
                         │
                    Worker do: block executes
                    (call: http → push topic: helpdesk:notifications)
                         │
                    Goal met: status==RESOLVED && notified==true
                         │
                    Case complete ✓
```

The playbook visualiser renders this entire flow as an interactive visual. Each binding is a step. Each worker execution is visible. The reviewer sees the behaviour without reading YAML.

---

## The developer's file set

```
helpdesk-apps/
  src/main/resources/
    org/apps/helpdesk/
      app.yaml                    # collections, schema, seed data, access
      helpdesk.page.yaml          # UI: dashboard, queue, detail, forms
      data/
        tickets.json              # 12 seed tickets
        agents.json               # 5 seed agents
    casehub/
      helpdesk-ticket.yaml        # case lifecycle: bindings, workers, goals
    playbooks/
      helpdesk-demo.yaml          # interactive demo scenario
  src/main/webui/                 # minimal TS (only if needed)
    pipeline-strategy.ts          # ticket → timeline mapping (if not YAML-expressible)
  pom.xml                         # runtime deps only, zero Java sources
```

### app.yaml — data model

Defines two collections with typed schemas and seed data:

```yaml
name: helpdesk
displayName: Helpdesk
version: 1.0.0

collections:
  tickets:
    fields:
      subject:      { type: string }
      description:  { type: string }
      category:     { type: enum, values: [HARDWARE, SOFTWARE, ACCESS, OTHER] }
      priority:     { type: enum, values: [LOW, MEDIUM, HIGH, URGENT] }
      status:       { type: enum, values: [OPEN, TRIAGED, ASSIGNED, RESOLVED, CLOSED] }
      customerRef:  { type: string }
      assigneeId:   { type: ref, collection: agents }
      resolution:   { type: string }
    seed: data/tickets.json

  agents:
    fields:
      name:        { type: string }
      email:       { type: string }
      speciality:  { type: enum, values: [HARDWARE, SOFTWARE, ACCESS, GENERAL] }
      active:      { type: boolean, default: true }
    seed: data/agents.json

access:
  read: authenticated
  write: [admin, developer]
```

### helpdesk-ticket.yaml — case lifecycle

The brain of the application. Defines reactive bindings with inline worker logic:

```yaml
dsl: 1.0.0
namespace: casehub-examples
name: helpdesk-ticket

capabilities:
  - name: classify-ticket
    input:
      jq: '{ subject: .subject, description: .description }'
    output:
      jq: '. + { status: "TRIAGED" }'

  - name: notify-customer
    input:
      jq: '{ customerRef: .customerRef, resolution: .resolution }'
    output:
      jq: '. + { notified: true }'

workers:
  - name: keyword-classifier
    capabilities: [classify-ticket]
    do:
      - classify:
          switch:
            - when: "${ .subject | test(\"laptop|monitor|keyboard|printer\"; \"i\") }"
              then:
                set: { category: HARDWARE, priority: HIGH }
            - when: "${ .subject | test(\"password|access|login|vpn\"; \"i\") }"
              then:
                set: { category: ACCESS, priority: LOW }
            - when: "${ .subject | test(\"install|update|license|ide\"; \"i\") }"
              then:
                set: { category: SOFTWARE, priority: MEDIUM }
            - default:
                set: { category: OTHER, priority: MEDIUM }

  - name: notification-sender
    capabilities: [notify-customer]
    do:
      - notify:
          call: http
          with:
            method: POST
            endpoint:
              uri: /api/push/publish/helpdesk:notifications
            body:
              to: "${ .customerRef }"
              message: "Your ticket '${ .subject }' has been resolved: ${ .resolution }"

bindings:
  - name: triage
    trigger:
      on: context-change
      when: "${ .status == \"OPEN\" and .category == null }"
    invoke: classify-ticket

  - name: assign-specialist
    trigger:
      on: context-change
      when: "${ .status == \"TRIAGED\" and .resolution == null }"
    human-task:
      title: "Resolve: ${ .subject }"
      candidate-groups:
        jq: >
          if .category == "HARDWARE" then ["hw-specialist"]
          elif .category == "SOFTWARE" then ["sw-specialist"]
          elif .category == "ACCESS" then ["access-specialist"]
          else ["general-specialist"]
          end

  - name: notify-resolution
    trigger:
      on: context-change
      when: "${ .status == \"RESOLVED\" and .notified == null }"
    invoke: notify-customer

goals:
  - name: ticket-resolved
    success:
      when: "${ .status == \"RESOLVED\" and .notified == true }"

completion:
  success:
    all-goals: met
```

### helpdesk.page.yaml — UI definition

Reuses the existing Pages helpdesk page YAML (from the Pages repo):
- Dashboard with KPI metrics (open, urgent, unassigned, resolved)
- Charts by category, priority, status
- Ticket queue with filter dropdowns
- Ticket detail form with auto-save
- New ticket form with schema validation
- Agent roster with speciality chart

### helpdesk-demo.yaml — playbook scenario

Extends the existing 4-step intake playbook into a full lifecycle demo:
1. Bootstrap classification rules
2. Submit a ticket via the form
3. Watch classification fire automatically (visualiser shows the binding)
4. See work item created for specialist
5. Claim and resolve the work item
6. Watch notification fire automatically
7. Verify on dashboard

Each step is visible in the playbook visualiser, including the case engine's reactive steps.

---

## Platform work

### 1. Case engine auto-discovery (casehub-engine)

**What:** Classpath scanner that discovers `*.yaml` case definitions and creates CaseHub CDI beans automatically at startup.

**Why:** Today every CaseHub requires a Java class:
```java
@ApplicationScoped
public class HelpDeskCaseHub extends YamlCaseHub {
    public HelpDeskCaseHub() { super("casehub/helpdesk-ticket.yaml"); }
}
```
For zero-Java apps, the engine must discover and register case definitions without application code.

**Approach:** A CDI extension or `@Startup` scanner that:
1. Scans `casehub/*.yaml` on the classpath
2. Parses each as a case definition
3. Registers a `YamlCaseHub` bean per definition
4. Wires workers via `FlowWorkerFunctionProvider` (already handles `do:` blocks)

**Size:** Small — the YAML parsing and worker wiring already exist. This is registration glue.

### 2. JSON collection SPI adapter (casehub-engine + data-json)

**What:** Standard `DataProvider` / `CaseInstanceCache` SPI implementation backed by `JsonCollectionStore`.

**Why:** The case engine needs a data persistence layer. JSON collections already store typed documents with optimistic concurrency. This is a standard SPI implementation, not a bridge — uniform data access.

**Approach:**
- Implement the case engine's persistence SPI using `JsonCollectionStore` APIs
- Case instances are JSON documents in a collection derived from the case definition name
- Field mappings use the `app.yaml` schema + case definition capability projections
- Optimistic concurrency via the existing `_version` column

**Size:** Medium — SPI implementation with field mapping and concurrency semantics.

### 3. Push topic publish endpoint (casehub-pages-push)

**What:** REST endpoint for publishing events to push topics: `POST /api/push/publish/{topic}`

**Why:** Workers' `do:` blocks use `call: http`. They need a REST endpoint to publish events to push topics. Currently, publishing is only available from server-side Java code.

**Approach:** Add a `PushPublishResource` with:
- `POST /api/push/publish/{topic}` — accepts JSON payload, broadcasts to topic subscribers
- Authentication/authorization consistent with existing push infrastructure

**Size:** Small — a thin REST wrapper over existing broadcast logic.

### 4. TS decorator framework (casehub-pages, new module)

**What:** TypeScript decorators providing CDI-like event-driven wiring: `@Observes('topic')`, `@Publish('topic')`, `@ApplicationScoped`.

**Why:** Web developers think in TypeScript, not YAML. The decorator framework makes Pages appealing to this audience while using the same underlying paradigm.

**Approach:**
- `@Observes(topic)` — decorator on methods. Subscribes to a push topic via WebSocket. When a message arrives on the topic, invokes the decorated method.
- `@Publish(topic)` — decorator on methods. After the method returns, publishes the return value to the specified push topic.
- `@ApplicationScoped` — class decorator. Ensures a single instance, created at app startup, destroyed on app teardown.
- Transport: WebSocket connection to the existing push infrastructure.
- Registration: a lightweight container that scans decorated classes and wires subscriptions at startup.

**Size:** Medium — decorator implementation + lightweight DI container + WebSocket integration.

### 5. Playbook visualiser — case binding rendering (casehub-pages)

**What:** The playbook visualiser must render case YAML bindings and worker `do:` blocks as visual steps in the flow.

**Why:** The YAML is simultaneously the runtime configuration and the visual model. For LLM-generated apps, the visualiser is the review surface. If bindings aren't visible, the reviewer misses the core behaviour.

**Approach:**
- Parse case definition YAML bindings as visualisable steps
- Each binding becomes a node in the flow: trigger condition → worker execution → outcome
- Worker `do:` blocks render as sub-steps (switch branches, HTTP calls)
- State transitions (OPEN → TRIAGED → ASSIGNED → RESOLVED) render as a timeline

**Size:** Medium — extends the existing visualiser's step model to understand case bindings.

---

## Side-by-side comparison

| Aspect | Java (`helpdesk/`) | YAML (`helpdesk-apps/`) |
|--------|-------------------|------------------------|
| Data model | Java records + enums (5 files) | `app.yaml` collections |
| Case lifecycle | Java dispatchers (4 files) | `helpdesk-ticket.yaml` `do:` blocks |
| Reactive bindings | Java CaseHub + augment() | YAML bindings section |
| Event handling | CDI `@Observes` (3 files) | Push topics from YAML workers |
| UI | Lit web components (3 TS files) | `helpdesk.page.yaml` |
| Demo bootstrap | Java `@PlaybookAction` (2 files) | YAML playbook steps |
| Push/WebSocket | Java endpoint + observer (4 files) | Built-in push infrastructure |
| MCP integration | Java interface + impl (6 files) | Declarative (TBD) |
| REST endpoints | Java resources (4 files) | Auto-generated from app.yaml |
| **Total** | ~20 Java + YAML + TS | ~5 YAML + seed JSON |

---

## Delivery sequence

### Phase 1 — Foundation (engine + pages changes)

1. JSON collection SPI adapter for case engine
2. Case YAML auto-discovery
3. Push topic publish endpoint

These unblock the YAML-only helpdesk.

### Phase 2 — YAML helpdesk example

4. Create `helpdesk-apps/` module with app.yaml, case YAML, page YAML, playbooks, seed data
5. Validate end-to-end: ticket creation → classification → work item → resolution → notification
6. Verify playbook visualiser renders the lifecycle flow

### Phase 3 — TS decorator framework

7. Implement `@Observes`, `@Publish`, `@ApplicationScoped` decorators
8. Wire to push topic WebSocket transport
9. Create a small TS example (or TS variant of helpdesk) demonstrating the decorator path

### Phase 4 — Polish

10. Side-by-side documentation comparing Java and YAML versions
11. Playbook visualiser enhancements for case binding rendering
12. LLM generation tutorial (prompt → YAML → visualise → review)

---

## Open questions

1. **MCP integration in YAML** — The Java helpdesk exposes MCP operations (`@McpDomain("helpdesk")`). Can MCP domains be declared in YAML? Or is MCP a Java-only extension point for now?

2. **Classification data source** — The Java version loads classification rules at bootstrap. The YAML version has them inline in the `switch:` block. Should classifications be a separate collection in app.yaml (editable at runtime) or inline in the case YAML (static)?

3. **Work item service** — The case YAML's `human-task` binding creates work items. Does the work item service need to be available as a REST endpoint that the JSON-collection-backed helpdesk can call, or does the case engine handle this internally?

4. **Chat inbound** — The Java version creates cases from chat messages via CDI events. The YAML version needs a REST endpoint for ticket creation that triggers the case engine. Is `POST /api/apps/helpdesk/tickets` sufficient, or does the case engine need a dedicated case-creation endpoint?

---

## References

- Pages helpdesk app.yaml: `casehub-pages/examples/server/src/main/resources/org/apps/helpdesk/app.yaml`
- Pages helpdesk page YAML: `casehub-pages/examples/server/src/main/resources/org/apps/helpdesk/helpdesk.page.yaml`
- Java helpdesk case definition: `examples/helpdesk/src/main/resources/casehub/helpdesk-ticket.yaml`
- Java helpdesk case hub: `examples/helpdesk/src/main/java/.../engine/HelpDeskCaseHub.java`
- FlowWorkerFunctionProvider: `casehub-engine` (handles `do:` blocks)
- Document processing example: `casehub-engine/schema/src/main/resources/examples/document-processing.yaml`
- Blog: "Pages becomes an app runtime" — `casehub-pages/docs/blog/2026-10-30-mdp01-pages-becomes-an-app-runtime.md`
- Pages push infrastructure: `casehub-pages-push`, `casehub-pages-push-runtime`
