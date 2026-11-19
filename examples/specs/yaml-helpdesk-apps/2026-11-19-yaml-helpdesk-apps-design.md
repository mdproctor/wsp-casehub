# YAML Helpdesk Apps — Design Spec

**Date:** 2026-11-19
**Status:** draft
**Scope:** casehubio/examples, casehub-engine, casehub-pages, casehub-eidos

## Vision

A complete helpdesk application — case lifecycle, LLM-based classification, eidos agent descriptors, reactive automation, dashboards, forms — defined entirely in YAML. Zero Java. Zero compilation. Not just a port of the existing helpdesk, but an **upgrade**: a simple but complete showcase for eidos agent identity, LLM classification, and the agentic dispatch pattern.

Three motivations drive this:

1. **Eidos + LLM + simulation showcase.** The helpdesk demonstrates a simple, complete agentic pattern: support agents defined as eidos descriptors, an LLM classifier agent, and capability-based routing. The agent manifest wires to either a real LLM or a simulator — every demo runs both ways. Simulator mode needs no API key and responds deterministically; LLM mode shows semantic classification. The switchable wiring also teaches the simulation pattern itself.

2. **LLM-generated apps with visual review.** An LLM generates YAML. The playbook visualiser renders the full lifecycle as an interactive flow. A human reviewer sees the behaviour, not the syntax. This scales where reading code or YAML does not.

3. **Web developer onboarding.** TS decorators (`@Observes`, `@Publish`) provide a familiar code-first authoring mode that compiles to the same runtime. YAML and TS are peer syntaxes — same paradigm, different audiences.

The Java helpdesk already demonstrates the core agentic pattern — workers as agents with capabilities, capability-based dispatch, dynamic routing. The YAML version expresses those same patterns declaratively, and upgrades the classification from keyword matching to LLM-based via eidos. Both remain alongside as a comparison: same agentic model, two authoring approaches.

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
                    (call: agent → LLM or simulator classifies)
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
    META-INF/eidos/
      descriptors.yaml            # eidos agent descriptors for support agents + classifier
      agent-manifest.yaml         # LLM provider config (which model, credentials)
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

The brain of the application. Defines reactive bindings with workers that use eidos agents:

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
  # LLM-based classifier — uses eidos agent descriptor for prompt + agent-config for LLM provider
  - name: ticket-classifier
    capabilities: [classify-ticket]
    agent: classifier-agent          # references eidos descriptor by slot
    do:
      - classify:
          call: agent                # invokes the eidos agent via configured LLM
          with:
            prompt: >
              Classify this support ticket into one category and priority.
              Categories: HARDWARE, SOFTWARE, ACCESS, OTHER
              Priorities: LOW, MEDIUM, HIGH, URGENT
              Ticket subject: ${ .subject }
              Ticket description: ${ .description }
            output-schema:
              category: { type: enum, values: [HARDWARE, SOFTWARE, ACCESS, OTHER] }
              priority: { type: enum, values: [LOW, MEDIUM, HIGH, URGENT] }

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
    # Eidos capability-based routing — matches agents by capability, not hardcoded groups
    agent-task:
      title: "Resolve: ${ .subject }"
      capability: "${ .category | ascii_downcase }-support"

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

### descriptors.yaml — eidos agent descriptors

Defines the agents with capabilities, disposition, and personality:

```yaml
# META-INF/eidos/descriptors.yaml
agents:
  - slot: classifier-agent
    name: Ticket Classifier
    capabilities:
      - name: ticket-classification
        domain: support-triage
        quality: high
        latency: low
    disposition:
      conflict-mode: analytical
      risk-appetite: conservative

  - slot: hw-specialist
    name: Carlos Rivera
    capabilities:
      - name: hardware-support
        domain: hardware-troubleshooting
    disposition:
      social-orientation: collaborative
      communication-style: methodical

  - slot: sw-specialist
    name: Jordan Lee
    capabilities:
      - name: software-support
        domain: software-troubleshooting
    disposition:
      social-orientation: independent
      communication-style: direct

  - slot: access-specialist
    name: Alex Thompson
    capabilities:
      - name: access-support
        domain: identity-access-management
    disposition:
      social-orientation: collaborative
      communication-style: patient
```

### agent-manifest.yaml — LLM / simulator provider config

The manifest wires agents to either a real LLM or a simulator. Every demo must support both modes — switchable via profile or config. Simulator mode runs without an API key, making demos self-contained. It also teaches the simulation pattern itself.

```yaml
# META-INF/eidos/agent-manifest.yaml
providers:
  # Real LLM — used in production or live demos
  - name: classifier-llm
    type: anthropic
    model: claude-haiku-5-5
    source:
      credential-ref: anthropic-api-key

  # Simulator — used for demos, testing, and teaching
  - name: classifier-sim
    type: simulator
    responses:
      classify-ticket:
        rules:
          - when: { subject-contains: [laptop, monitor, keyboard, printer] }
            then: { category: HARDWARE, priority: HIGH }
          - when: { subject-contains: [password, access, login, vpn] }
            then: { category: ACCESS, priority: LOW }
          - when: { subject-contains: [install, update, license, ide] }
            then: { category: SOFTWARE, priority: MEDIUM }
          - default: { category: OTHER, priority: MEDIUM }

# Active provider — switch between LLM and simulator
# Set via config property or Quarkus profile:
#   -Dagent.provider=classifier-llm   (real LLM)
#   -Dagent.provider=classifier-sim   (simulator, default for demos)
active: classifier-sim
```

This dual-wiring pattern means:
- **Demo mode** (default): simulator responds deterministically, no API key needed, instant responses
- **Live mode**: real LLM classifies semantically, requires API key, shows the LLM upgrade over keywords
- **Teaching**: the simulator config itself is a teaching tool — users see how simulation rules map to LLM behaviour, understand what the simulator replaces

### helpdesk.page.yaml — UI definition

Reuses the existing Pages helpdesk page YAML (from the Pages repo):
- Dashboard with KPI metrics (open, urgent, unassigned, resolved)
- Charts by category, priority, status
- Ticket queue with filter dropdowns
- Ticket detail form with auto-save
- New ticket form with schema validation
- Agent roster with speciality chart

### helpdesk-demo.yaml — playbook scenario + integration test

The playbook serves dual purpose: **teaching** (visual walkthrough in browser) and **testing** (headless execution with assertions as integration tests).

```yaml
# playbooks/helpdesk-demo.yaml
schema: client
version: 1.0

steps:
  # 1. Bootstrap
  - name: bootstrap-classifications
    action: bootstrap-classifications
    data: { ... }

  # 2. Submit ticket via UI (ARIA selectors — tests accessibility as a side effect)
  - name: submit-ticket
    fill: { role: textbox, name: "Subject", value: "Laptop won't boot" }
  - name: select-priority
    select: { role: combobox, name: "Priority", value: "HIGH" }
  - name: click-submit
    click: { role: button, name: "Submit" }

  # 3. Assert: ticket classified correctly (business logic assertion)
  - name: verify-classification
    verify:
      endpoint: /api/apps/helpdesk/tickets
      expect:
        - match: { subject: "Laptop won't boot" }
          assert:
            category: HARDWARE
            priority: HIGH
            status: TRIAGED

  # 4. Assert: work item created for specialist
  - name: verify-work-item
    verify:
      endpoint: /api/work/items/list-all
      expect:
        - match: { title-contains: "Laptop won't boot" }
          assert:
            candidate-groups: [hw-specialist]
            status: PENDING

  # 5. Claim and resolve (ARIA selectors)
  - name: claim-work-item
    click: { role: button, name: "Claim" }
  - name: resolve-ticket
    fill: { role: textbox, name: "Resolution", value: "Replaced battery" }
  - name: click-resolve
    click: { role: button, name: "Resolve" }

  # 6. Assert: notification sent
  - name: verify-notification
    verify:
      endpoint: /api/push/topics/helpdesk:notifications
      expect:
        - assert:
            to: { not-empty: true }
            message: { contains: "Laptop won't boot" }

  # 7. Assert: case complete
  - name: verify-case-complete
    verify:
      endpoint: /api/apps/helpdesk/tickets
      expect:
        - match: { subject: "Laptop won't boot" }
          assert:
            status: RESOLVED
            notified: true
```

**Two execution modes:**

| Mode | How | Purpose |
|------|-----|---------|
| **Teaching** | Browser with playbook visualiser | User watches each step execute, sees bindings fire, understands the flow |
| **Testing** | Headless Playwright (Quarkus integration) via CI | Same playbook runs automated, `verify:` steps become pass/fail assertions, produces test report |

Playbook steps use **ARIA-based selectors** (`role: button, name: "Submit"`) instead of CSS selectors. This means:
- Tests verify accessibility as a side effect — if ARIA roles are broken, the test fails
- Selectors are resilient to UI structure changes
- Demonstrates accessible-first testing as a best practice

The `verify:` steps are the key addition — in teaching mode they show green checkmarks in the visualiser; in testing mode they're assertion failures that break the build.

This closes the LLM-generated apps loop: **generate → review → test**, all from YAML. An LLM generates the app YAML and the playbook. The visualiser lets you review the behaviour. Headless execution verifies it actually works. No separate test harness needed.

### Three-way viewer — YAML / visual / split

The helpdesk demo currently shows YAML in a text viewer. With the graphical playbook viewer now available, the app needs three viewing modes:

| Mode | What the user sees |
|------|--------------------|
| **YAML** | Raw YAML text — the case definition, playbook steps, agent descriptors |
| **Visual** | Graphical flow — bindings as nodes, state transitions as edges, agent dispatch visible |
| **Split** | Both side by side — YAML on the left, visual on the right, linked |

**Linked navigation:** In split mode, clicking a visual step highlights the corresponding YAML block. Editing YAML live-updates the visual. This makes the relationship between declaration and behaviour explicit.

**Applies to all YAML artifacts:**
- Case definition YAML → visual shows binding flow, state machine, goals
- Playbook YAML → visual shows step sequence, verify checkpoints, branch points
- Eidos descriptors → visual shows agent capabilities, disposition axes
- Agent manifest → visual shows provider wiring (LLM vs simulator)

**Why this matters:**
- **Learning:** "what does this YAML produce?" — see it instantly
- **LLM review:** generated YAML alongside its visual flow — catch errors visually
- **Debugging:** trace a failure from visual step back to YAML source
- **Teaching simulation:** in split mode, see simulator rules alongside the visual flow they produce

---

## Platform work

### 0. Eidos agent dispatch from case YAML (casehub-eidos + casehub-engine)

**What:** Workers in case YAML reference eidos agents by slot. At dispatch time, the engine resolves the agent via `AgentRegistry`, uses the descriptor for prompt generation (`SystemPromptRenderer`), and calls the LLM via agent-config.

**Why:** The existing `FlowWorkerFunctionProvider` supports `call: http`. We need a `call: agent` variant that calls an eidos-registered agent. This is the bridge between case YAML workers and the eidos/LLM stack.

**Approach:**
- New `agent:` field on workers — references an eidos descriptor slot
- New `call: agent` step type in `do:` blocks — invokes the referenced agent via configured LLM
- Agent-task bindings use eidos capability matching (`AgentRegistry.find()`) instead of hardcoded candidate groups
- `CapabilityHealth.probe()` for agent availability at dispatch time

**Size:** Medium — new worker function provider + binding resolver, but eidos and agent-config already exist.

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
| Classification | Java keyword matching (simple agent) | LLM via eidos agent + `call: agent` |
| Agent identity | Java workers with capabilities | Eidos descriptors (capabilities, disposition) |
| Agent routing | Dynamic candidate groups (JQ) | Eidos capability-based matching |
| LLM config | N/A (keyword-based) | `agent-manifest.yaml` |
| Case lifecycle | Java dispatchers (4 files) | YAML `do:` blocks |
| Reactive bindings | Java CaseHub + augment() | YAML bindings section |
| Event handling | CDI `@Observes` (3 files) | Push topics from YAML workers |
| UI | Lit web components (3 TS files) | `helpdesk.page.yaml` |
| Demo bootstrap | Java `@PlaybookAction` (2 files) | YAML playbook steps |
| Push/WebSocket | Java endpoint + observer (4 files) | Built-in push infrastructure |
| MCP integration | Java interface + impl (6 files) | Declarative (TBD) |
| REST endpoints | Java resources (4 files) | Auto-generated from app.yaml |
| **Total** | ~20 Java (agentic pattern, keyword classifier) | ~7 YAML + seed JSON (same pattern, LLM classifier) |

---

## Delivery sequence

### Phase 1 — Foundation (engine + eidos + pages)

1. Eidos agent dispatch from case YAML (`call: agent`, `agent-task:` bindings)
2. JSON collection SPI adapter for case engine
3. Case YAML auto-discovery
4. Push topic publish endpoint

These unblock the YAML-only helpdesk with full agentic capabilities.

### Phase 2 — YAML helpdesk example

5. Create `helpdesk-apps/` module: app.yaml, case YAML, page YAML, eidos descriptors, agent manifest, playbooks, seed data
6. Validate end-to-end: ticket creation → LLM classification → eidos-routed work item → resolution → notification
7. Verify playbook visualiser renders the lifecycle flow including agent dispatch

### Phase 3 — TS decorator framework

8. Implement `@Observes`, `@Publish`, `@ApplicationScoped` decorators
9. Wire to push topic WebSocket transport
10. Create a small TS example (or TS variant of helpdesk) demonstrating the decorator path
11. Demonstrate YAML and TS expressing the same behaviour — same runtime, two syntaxes

### Phase 4 — Viewer + testing

12. Three-way viewer (YAML / visual / split) with linked navigation
13. Headless playbook runner (Playwright, Quarkus integration) — playbooks as CI integration tests
14. ARIA-based selectors in playbook steps

### Phase 5 — Polish

15. Side-by-side documentation comparing Java and YAML versions
16. Playbook visualiser enhancements for case binding + agent rendering
17. LLM generation tutorial (prompt → YAML → visualise → review)

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
