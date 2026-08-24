# CaseHub Automation & Scenario Format — v3 Design

**Issue:** casehubio/parent#409
**Date:** 2026-08-24
**Status:** Draft

## Overview

This spec defines the canonical YAML format for CaseHub automations and scenarios. An **automation** is a sequence of steps executed by one or more executors — seeding data, running smoke tests, driving workflows. A **scenario** is an automation with a narrative overlay (chapters and sections) for human-paced execution — demos, walkthroughs, training.

Both share the same execution engine. The format is the contract between YAML authors (humans and LLMs) and executors (browser, backend services, third-party integrations).

### Design principles

- **GraphQL-first.** Every CaseHub service exposes GraphQL via `@McpDomain` + `@PlatformQuery`/`@PlatformMutation`. The format uses GraphQL as the canonical server action type. (D1)
- **Three action categories.** ARIA for browser UI, GraphQL for CaseHub services, HTTP for third-party APIs. No "delivery modes." (D2)
- **ARIA by role + name.** Frontend elements are referenced by ARIA role and accessible name. No CSS selectors. (D3)
- **Distributed fragment execution.** The orchestrator partitions the YAML into fragments per executor. Each executor runs its fragment autonomously. (D4)
- **Steps are dispatch units.** All commands in a step run on one executor. Switching executors means a new step. (D8)

## Document Structure

A YAML document has a top-level `scenario` name and optional metadata, followed by the execution body at one of three entry points:

```yaml
scenario: <name>                    # required — unique identifier
description: <text>                 # optional — human-readable summary
speed: <number>                     # optional — inter-step delay multiplier (default: 1)
on-error: stop | continue | pause   # optional — error mode (default: stop)

# Entry point — exactly one of:
chapters: [...]                     # full narrative hierarchy
sections: [...]                     # mid-level grouping
steps: [...]                        # flat automation
```

The three entry points are mutually exclusive. Chapters contain sections; sections contain steps; steps contain commands. This layering is flexible — simple automations skip straight to steps, full demos use the whole hierarchy. (D6)

### Chapters

Optional narrative grouping for human-paced scenarios. Chapters have no execution semantics — they group sections for navigation ("run to chapter 3") and pacing.

```yaml
chapters:
  - label: "Customer Reports Issue"
    sections:
      - label: "Customer sends message"
        steps: [...]
      - label: "System classifies ticket"
        steps: [...]
  - label: "Agent Resolves Issue"
    sections: [...]
```

### Sections

Group steps within a chapter (or at the top level if no chapters). Like chapters, sections are presentation structure — the executor pauses between sections when a human is pacing.

### Steps

The execution unit. A step groups commands that run on a single executor. All commands in a step execute sequentially on the step's target.

```yaml
steps:
  - label: "Submit support request"    # required — human-readable description
    name: submit                        # optional — machine identifier for variable interpolation
    target: browser                     # required — which executor runs this step
    commands:                           # required — at least one command
      - action: fill
        element: {role: textbox, name: "Subject"}
        value: "Laptop won't boot"
      - action: click
        element: {role: button, name: "Submit"}
```

**Fields:**

| Field | Required | Description |
|-------|----------|-------------|
| `label` | yes | Human-readable description, unique within its parent |
| `name` | no | Machine identifier for variable interpolation (`${name.field}`). Required if later steps reference this step's results. Must be unique within the scenario. |
| `target` | yes | Executor that runs this step (`browser`, service name, etc.) |
| `commands` | yes | Ordered list of commands to execute |

`target` routes the step to a named executor. The orchestrator validates that all targets have registered executors before starting. (D8, D9)

## Commands

The atomic action unit. Every command has an `action` field as the type discriminator and action-specific parameters. (D7)

```yaml
commands:
  - action: <action-name>
    # ... action-specific fields
```

### ARIA Actions

Browser UI automation using ARIA roles and accessible names. The `element` field identifies the target DOM element. (D3, D9)

| Action | Fields | Description |
|--------|--------|-------------|
| `navigate` | `value: <url-or-hash>` | Navigate to URL or hash route |
| `click` | `element: {role, name}` | Click an element |
| `fill` | `element: {role, name}`, `value: <text>` | Type text into an input |
| `select` | `element: {role, name}`, `value: <option>` | Select an option from a dropdown |
| `expand` | `element: {role, name}` | Expand a collapsible element |
| `collapse` | `element: {role, name}` | Collapse an expanded element |
| `assert` | `element: {role, name}`, `state: {<aria-attr>: <value>}` | Assert element state |
| `wait` | `element: {role, name}`, `state: {<aria-attr>: <value>}`, `timeout: <ms>` | Poll until element reaches state |

**Element reference:**

```yaml
element:
  role: button          # ARIA role
  name: "Submit"        # Accessible name
```

For scoped lookups (element within a container):

```yaml
element:
  role: button
  name: "Submit"
  within:
    role: dialog
    name: "Confirmation"
```

`within` is recursive — any depth of scoping is supported.

**Examples:**

```yaml
# Navigate to a page
- action: navigate
  value: "/helpdesk/intake"

# Fill a text field
- action: fill
  element: {role: textbox, name: "Subject"}
  value: "Laptop won't boot"

# Click a button inside a dialog
- action: click
  element:
    role: button
    name: "Confirm"
    within: {role: dialog, name: "Are you sure?"}

# Assert an alert is visible
- action: assert
  element: {role: alert, name: "Ticket created"}
  state: {aria-hidden: false}

# Wait for a spinner to finish
- action: wait
  element: {role: button, name: "Submit"}
  state: {aria-busy: false}
  timeout: 5000
```

### GraphQL Actions

CaseHub server operations via the platform's GraphQL API. The `domain` field routes to the correct `@McpDomain` resolver; `operation` names the `@PlatformQuery` or `@PlatformMutation` method. (D1)

```yaml
- action: graphql
  domain: <mcp-domain>        # @McpDomain value (e.g. "connectors")
  operation: <method-name>    # @PlatformQuery/@PlatformMutation method name
  params:                     # operation parameters (key-value)
    platform: "slack"
    sender: "Alice"
    message: "My laptop won't boot"
```

**With await** — poll for a condition on the result:

```yaml
- action: graphql
  domain: connectors
  operation: injectChat
  params:
    platform: "slack"
    sender: "Alice"
    message: "My laptop won't boot"
  await:
    match:
      category: "HARDWARE"
    timeout: 30000
    interval: 500
```

The executor calls the GraphQL mutation, then polls at `interval` ms until the result matches `match` or `timeout` ms elapse. `interval` defaults to 1000ms.

**Dispatch:** The executor builds an HTTP POST to the service's `/graphql` endpoint. The `GraphQLResolverProcessor` generates resolvers from SPI annotations — the YAML author writes `domain: connectors, operation: injectChat` and the platform routes to the right generated resolver.

### HTTP Actions

Third-party REST calls for external integrations.

```yaml
- action: http
  method: POST                            # HTTP method
  url: "https://api.example.com/notify"   # target URL
  headers:                                # optional headers
    Authorization: "Bearer ${token}"
    Content-Type: "application/json"
  body:                                   # optional request body
    ticketId: "${create.ticketId}"
    status: "resolved"
```

HTTP actions support variable interpolation in `url`, `headers`, and `body` fields.

### Custom Actions

Any action name not in the well-known set (ARIA actions, `graphql`, `http`) is a **custom action** handled by the target executor. Custom actions map to `@ScenarioAction` CDI handlers on the service.

```yaml
# YAML
- action: create-ticket
  data:
    subject: "Laptop won't boot"
    category: "HARDWARE"

# Java handler on the target service
@ScenarioAction("create-ticket")
Map<String, Object> createTicket(ActionContext ctx) {
    var ticket = ticketService.create(ctx.data("subject"), ctx.data("category"));
    return Map.of("ticketId", ticket.id().toString());
}
```

Custom actions use a `data` field for key-value parameters. The executor's `@ScenarioAction` registry maps the action name to a handler method. The handler's return value is available for variable interpolation.

**Well-known action names** (reserved — cannot be used as custom actions):
`navigate`, `click`, `fill`, `select`, `expand`, `collapse`, `assert`, `wait`, `graphql`, `http`

## Variable Interpolation

Steps that produce results can be referenced by later steps using `${stepName.field.path}` syntax. (D11)

```yaml
steps:
  - label: "Create ticket"
    name: create
    target: helpdesk
    commands:
      - action: create-ticket
        data: {subject: "Laptop won't boot"}
        # handler returns: {ticketId: "T-001", status: "OPEN"}

  - label: "Verify classification"
    name: verify
    target: helpdesk
    commands:
      - action: check-status
        data:
          ticketId: "${create.ticketId}"
```

**Rules:**
- Step `name` is the namespace — only named steps can be referenced
- Dot-path navigates nested result objects: `${create.details.category}`
- Interpolation happens in `value`, `data`, `params`, `url`, `headers`, `body`, and `await.match` fields
- Unknown step reference throws with available step names listed
- Step names must be unique within the scenario
- Regex: `\$\{([^}]+)}`

## Error Handling

The `on-error` field at the scenario level controls failure behaviour. (D10)

| Mode | Behaviour | Use case |
|------|-----------|----------|
| `stop` (default) | Abort all executors immediately | Unattended automations — fail-fast |
| `continue` | Skip the failed step's dependents transitively | Resilient pipelines — run what you can |
| `pause` | Freeze all executors for operator intervention | Human-paced demos — let the operator decide |

**Dependency tracking for `continue` mode:** A step B depends on step A if B references A's results via variable interpolation (`${A.field}`). When A fails with `on-error: continue`, B is skipped. If C depends on B, C is also skipped (transitive).

**Step-level results on failure:**
- Failed step: `{ok: false, error: "<message>"}`
- Remaining commands in the step after the failure: skipped
- Await timeout: `{ok: false, error: "await timed out"}`

## Speed and Pacing

The `speed` field controls inter-step delay for human-paced scenarios.

```yaml
scenario: helpdesk-demo
speed: 1          # 1 = normal (1000ms between steps), 2 = double speed, 0.5 = half speed
on-error: pause
chapters: [...]
```

- Delay between steps: `1000 / speed` ms
- `speed: 0` is invalid (would mean infinite delay)
- Executors apply the delay between steps, not between commands within a step
- Speed can be adjusted at runtime via control messages (pause, resume, step, speed)

For unattended automations, omit `speed` — the default (1) applies a 1-second inter-step delay. Set `speed` to a high value for fast execution, or control via runtime messages.

## Dispatch Protocol

The orchestrator partitions the scenario into fragments per executor and distributes them over WebSocket. (D4)

### Fragment partitioning

1. Walk the step list (flattened from chapters/sections)
2. Group consecutive steps with the same `target` into a fragment
3. Send each fragment to its target executor via `dispatch-sequence`

### Wire protocol

```
Executor → Orchestrator: executor-register
  {op: "executor-register", name: "helpdesk", actions: ["create-ticket", "resolve-ticket"]}

Orchestrator → Executor: dispatch-sequence
  {op: "dispatch-sequence", sessionId: "s-001", executorId: "helpdesk",
   steps: [...], speed: 1.0, paused: false}

Executor → Orchestrator: step-result
  {op: "step-result", sessionId: "s-001", stepName: "create", ok: true,
   result: {ticketId: "T-001"}}

Orchestrator → Executor: executor-control
  {op: "executor-control", sessionId: "s-001", command: "pause|resume|step|speed",
   value: <number for speed>}
```

**Batching rules:**
- Untriggered steps with the same target in the same section are batched into one `dispatch-sequence`
- Steps that depend on results from other executors are dispatched individually after the dependency resolves
- New sequences arriving while one is running are queued and appended

### Browser-only mode

When no backend orchestrator is present, the browser executor runs the full YAML directly — parsing it locally, managing its own step sequencing, and executing ARIA commands against the DOM. GraphQL commands go via HTTP to the service's `/graphql` endpoint. This is the same execution model without the distribution layer.

## Reconciliation

### Delete (dead code)

Format A's parser and execution code is orphaned — no production code path references it. Delete:

- `pages/backend/scenario/` — `ScenarioParser.java`, `Scenario.java`, `ScenarioStep.java` (sealed interface with AriaStep/GraphQLStep/SimulatedStep), `AriaTarget.java`, `AwaitCondition.java`
- `pages/backend/scenario/src/test/` — `ScenarioParserTest.java` and all test YAML files
- `pages/backend/cross-parser-test/` — cross-parser round-trip tests for the old format

### Keep and adapt

- `pages/backend/scenario-runtime/VariableContext.java` — variable interpolation engine. Proven, matches D11's syntax. Adapt to the new command structure.
- `pages/backend/scenario-runtime/GraphQLDispatcher.java` — HTTP-to-GraphQL dispatch. Adapt to use `domain` + `operation` from the new format.
- `pages/backend/scenario-runtime/ScenarioExecutor.java` — sequential execution with variable context. Refactor for the new step/command hierarchy.
- `pages/backend/scenario-runtime/AriaDispatcher.java` — ARIA command dispatch to browser via push wire. Adapt field names (`element` instead of `target`).

### Supersede

- `pages/packages/pages-aria/src/scenario/types.ts` — TypeScript types still reference Format A's flat structure with `delivery` field. Rewrite to match this spec.
- `parent/docs/platform/scenario-format.md` — v1 spec. Replace with this spec once landed.
- `specs/issue-409-scenario-format/2026-08-24-scenario-format-v2-design.md` — v2 spec (discarded after review). Delete.

## Complete Examples

### Minimal automation — seed data

```yaml
scenario: seed-tickets
on-error: stop
steps:
  - label: "Create hardware ticket"
    name: hw
    target: helpdesk
    commands:
      - action: create-ticket
        data:
          subject: "Laptop won't boot"
          category: "HARDWARE"
          priority: "HIGH"

  - label: "Create software ticket"
    name: sw
    target: helpdesk
    commands:
      - action: create-ticket
        data:
          subject: "Excel crashes on startup"
          category: "SOFTWARE"
          priority: "MEDIUM"
```

### Full scenario — helpdesk demo

```yaml
scenario: helpdesk-demo
description: "IT help desk — customer reports issue through to resolution"
speed: 1
on-error: pause

chapters:
  - label: "Customer Reports Issue"
    sections:
      - label: "Customer submits support request"
        steps:
          - label: "Navigate to intake form"
            target: browser
            commands:
              - action: navigate
                value: "/helpdesk/intake"

          - label: "Fill out and submit the form"
            target: browser
            commands:
              - action: fill
                element: {role: textbox, name: "Subject"}
                value: "Laptop won't boot after update"
              - action: fill
                element: {role: textbox, name: "Description"}
                value: "Updated Windows last night, now stuck on blue screen"
              - action: select
                element: {role: combobox, name: "Category"}
                value: "HARDWARE"
              - action: click
                element: {role: button, name: "Submit"}

          - label: "Verify submission confirmation"
            target: browser
            commands:
              - action: assert
                element: {role: alert, name: "Ticket created"}
                state: {aria-hidden: false}

      - label: "System processes the ticket"
        steps:
          - label: "Inject chat message for classification"
            name: inject
            target: helpdesk
            commands:
              - action: graphql
                domain: connectors
                operation: injectChat
                params:
                  platform: "web-form"
                  sender: "alice.chen@example.com"
                  message: "Laptop won't boot after update"
                await:
                  match:
                    category: "HARDWARE"
                  timeout: 30000

  - label: "Agent Resolves Issue"
    sections:
      - label: "Agent picks up the ticket"
        steps:
          - label: "Navigate to agent dashboard"
            target: browser
            commands:
              - action: navigate
                value: "/helpdesk/dashboard"
              - action: click
                element:
                  role: row
                  name: "Laptop won't boot after update"

          - label: "Assign to current agent"
            name: assign
            target: helpdesk
            commands:
              - action: graphql
                domain: helpdesk
                operation: assignTicket
                params:
                  ticketId: "${inject.ticketId}"
                  agentId: "agent-001"

      - label: "Agent resolves and closes"
        steps:
          - label: "Add resolution notes"
            target: browser
            commands:
              - action: fill
                element: {role: textbox, name: "Resolution"}
                value: "Rolled back Windows update, laptop boots normally"
              - action: click
                element: {role: button, name: "Resolve"}

          - label: "Verify resolution"
            target: browser
            commands:
              - action: assert
                element: {role: status, name: "Ticket status"}
                state: {aria-current: "true"}
              - action: wait
                element: {role: button, name: "Resolve"}
                state: {aria-disabled: "true"}
                timeout: 3000
```

### HTTP integration — notify external system

```yaml
scenario: notify-slack
on-error: stop
steps:
  - label: "Create ticket"
    name: create
    target: helpdesk
    commands:
      - action: create-ticket
        data:
          subject: "Server disk full"
          priority: "CRITICAL"

  - label: "Notify Slack channel"
    target: browser
    commands:
      - action: http
        method: POST
        url: "https://hooks.slack.com/services/T00/B00/xxx"
        headers:
          Content-Type: "application/json"
        body:
          text: "Critical ticket created: ${create.ticketId} — Server disk full"
```

## References

- `pages/backend/scenario-runtime/src/main/java/.../VariableContext.java` — variable interpolation implementation
- `pages/backend/scenario-runtime/src/main/java/.../GraphQLDispatcher.java` — GraphQL HTTP dispatch
- `pages/backend/scenario-runtime/src/main/java/.../ScenarioExecutor.java` — step execution engine
- `pages/backend/scenario/src/test/resources/scenarios/*.yaml` — Format A test YAML (to be deleted)
- `specs/issue-408-scenario-engine/2026-08-20-distributed-executor-protocol-design.md` — Format B distributed executor protocol spec
- `platform/graphql-generator/.../GraphQLResolverProcessor.java` — annotation processor generating GraphQL resolvers
- `platform/platform-api/.../McpDomain.java`, `PlatformQuery.java`, `PlatformMutation.java` — SPI annotations
- `pages/packages/pages-aria/src/scenario/types.ts` — TypeScript types (to be rewritten)
- D1–D11 decisions: `specs/issue-409-scenario-format/decisions.md`
- Hybrid shorthand evaluation: casehubio/parent#423
