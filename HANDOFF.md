# Session Handover — issue-409-scenario-format

**Branch:** `issue-409-scenario-format` (project + workspace)
**Issue:** casehubio/parent#409 — Scenario format specification — YAML schema and semantics
**Epic:** casehubio/parent#408 — Cross-platform Scenario Engine

## Last Session

Brainstormed a v2 scenario format spec with GraphQL-first action model, ARIA-based
UI automation, and distributed fragment execution. Wrote a full design spec, then ran
a light design review that found 6 HIGH-priority issues — the spec was written without
visibility into the pages repo implementation. **The spec needs to be rewritten from
scratch — design the best architecture first, then propose how to get there.**

## Design Decisions (still valid — carry forward)

These decisions were confirmed by the user and remain the design intent:

- **D1:** GraphQL is the canonical API surface for MCP tooling and scenario automation. REST remains alongside it for external integration. Every CaseHub service must expose GraphQL — enforced at build time.
- **D2:** Three action types — GraphQL (server), ARIA (frontend), HTTP (third-party). No "delivery modes."
- **D3:** Direct ARIA references (role + accessible name). No CSS selectors.
- **D4:** Distributed execution — backend sends YAML script fragments to executors, batching as much as possible per executor boundary. Each executor runs its fragment locally.
- **D5:** YAML scenarios submitted to execution server via GraphQL mutation.

## User Guardrails (MANDATORY — read before writing anything)

1. **Design the best architecture first** — do NOT constrain the spec by what code currently exists. The spec should propose the ideal format, execution model, and capabilities. Compactness matters — less verbose is better. The scope is the whole system (YAML format, execution model, dispatch protocol), not just the YAML syntax.
2. **Do NOT read the v1 spec** (`parent/docs/platform/scenario-format.md`) — it will anchor you to the old delivery-mode model (rest/ui-form/simulated) which is superseded. The design decisions below already capture what was wrong with it.
3. **Understand what exists for reconciliation** — read the two implemented formats (A and B below) and the distributed executor protocol to understand what's running. Present findings to the user: what are they, why do they exist, should they unify? But design the ideal first, then propose migration.
4. **Distributed invocation model** — the coordinator sends YAML scripts to executors, not individual commands. It should batch as much as possible for each executor's boundary. The current implementation sends structured JSON `dispatch-sequence` messages over WebSocket — reconcile with the user's intent (YAML fragment distribution).

## Existing Formats (read for reconciliation, not as constraints)

**Format A — early parser** (`backend/scenario/` module):
Flat steps, ARIA shorthand inline, GraphQL via `delivery: graphql` + `domain` + `operation`.
Parser: `ScenarioParser.java`, executor: `ScenarioExecutor.java`, dispatcher: `GraphQLDispatcher.java`.
Test YAML: `backend/scenario/src/test/resources/scenarios/*.yaml`.
**Possibly orphaned** — the distributed executor protocol (Format B) may have superseded
this parser. Investigate whether anything still uses it or if it's dead code.

**Format B — distributed executor protocol** (spec for #418, production):
Hierarchical (chapters → sections → steps → commands), `target` field for executor routing,
`@ScenarioAction` CDI handlers on services, `dispatch-sequence` WebSocket messages.
Spec: `pages/wksp/specs/issue-408-scenario-engine/2026-08-20-distributed-executor-protocol-design.md`
Blog: `2026-08-21-mdp01-protocol-to-proof.md`, `2026-08-20-mdp02-distributed-executor-protocol.md`
This is what's running in production.

**Do NOT read:** `parent/docs/platform/scenario-format.md` (v1 spec — superseded, will anchor to old model).

**Platform GraphQL (for D1 context):**
- `platform/graphql-generator/.../GraphQLResolverProcessor.java`
- Annotations: `@McpDomain`, `@PlatformQuery`, `@PlatformMutation`

**TypeScript types:**
- `packages/pages-aria/src/scenario/types.ts`

## Review Findings Summary

Full review: `/Users/mdproctor/reviews/casehub-parent,/issue-409-scenario-format-v2-20260824-043729/`

HIGH:
- **R1-02:** Distributed execution model doesn't match reality (WebSocket push wire, not YAML fragments)
- **R1-03:** Three existing YAML formats — spec introduced a fourth without reconciling
- **R1-04:** Missing `domain` routing field for multi-service GraphQL dispatch
- **R1-05:** `injectMessage` mutation doesn't exist — simulated uses `@ScenarioAction`
- **R1-06:** `@ScenarioAction` service executor architecture ignored
- **R1-07:** Production uses chapters/sections/steps/commands hierarchy

MEDIUM insights worth carrying:
- **R1-08:** Variable syntax: `${stepName.field}` (existing) not `${steps.name.result.field}`
- **R1-09:** ARIA vocabulary must include `assert`, `expand`, `collapse` from existing code
- **R1-10:** Error model missing — v1 had a strong one, carry forward
- **R1-18:** Single `graphql:` key simpler than separate `mutation:`/`query:`

## Workspace Artifacts

- `specs/issue-409-scenario-format/decisions.md` — 5 decisions (carry forward)
- `specs/issue-409-scenario-format/2026-08-24-scenario-format-v2-design.md` — DISCARD, rewrite
- `specs/issue-409-scenario-format/pipeline.state` — reset to CONTEXT_GATHERING

## Immediate Next Step

Start brainstorming from scratch in a fresh session. Design the ideal architecture first (D1–D5 as intent), then read Formats A and B for reconciliation. Present the existing formats to the user before proposing a canonical format. Investigate whether Format A's parser is orphaned code.
