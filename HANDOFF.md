# Session Handover — issue-409-scenario-format

**Branch:** `issue-409-scenario-format` (project + workspace)
**Issue:** casehubio/parent#409 — Scenario format specification — YAML schema and semantics
**Epic:** casehubio/parent#408 — Cross-platform Scenario Engine

## Last Session

Brainstormed a v2 scenario format spec with GraphQL-first action model, ARIA-based
UI automation, and distributed fragment execution. Wrote a full design spec, then ran
a light design review that found 6 HIGH-priority issues — the spec describes an
architecture that doesn't match the existing implementation in the pages repo.
**The spec needs to be rewritten from scratch, grounded in the actual codebase.**

## Design Decisions (still valid — carry forward)

These decisions were confirmed by the user and remain the design intent:

- **D1:** GraphQL is the canonical API surface for MCP tooling and scenario automation. REST remains alongside it for external integration. Every CaseHub service must expose GraphQL — enforced at build time.
- **D2:** Three action types — GraphQL (server), ARIA (frontend), HTTP (third-party). No "delivery modes."
- **D3:** Direct ARIA references (role + accessible name). No CSS selectors.
- **D4:** Distributed execution — backend sends YAML script fragments to executors, batching as much as possible per executor boundary. Each executor runs its fragment locally.
- **D5:** YAML scenarios submitted to execution server via GraphQL mutation.

## User Guardrails (MANDATORY — read before writing anything)

1. **Improve but reconcile** — the new spec may improve on the existing YAML format (more compactness is better), but it must reconcile with what exists, not ignore it.
2. **Understand the three existing formats** — there are reportedly three YAML format variants already in the codebase. Identify what they are, why they exist, and whether they should be unified or each serves a distinct purpose. Present findings to the user before proposing a canonical format.
3. **Distributed invocation model** — the coordinator sends YAML scripts to executors, not individual commands. It should batch as much as possible for each executor's boundary. Understand how this currently works (the reviewer found WebSocket push wire dispatch with `dispatch-sequence` protocol messages) and reconcile with the user's intent (YAML fragment distribution).

## Files to Read First (from review findings)

The previous session had no visibility into the pages repo implementation. Read these before writing anything:

**Pages repo — existing scenario implementation:**
- `backend/scenario/src/main/java/io/casehub/pages/scenario/ScenarioParser.java`
- `backend/scenario/src/main/java/io/casehub/pages/scenario/ScenarioStep.java` — sealed interface: `AriaStep`, `GraphQLStep`, `SimulatedStep`
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/ScenarioExecutor.java`
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/GraphQLDispatcher.java` — uses `domain` for routing
- `backend/scenario-runtime/src/main/java/io/casehub/pages/scenario/runtime/VariableContext.java` — `${stepName.field}` syntax
- `backend/scenario/src/test/resources/scenarios/*.yaml` — existing test YAML (Format A)
- `packages/pages-aria/src/scenario/types.ts` — TypeScript types

**Distributed executor protocol:**
- Spec for parent#418 in pages workspace specs
- Blog: `2026-08-23-mdp01-scenario-engine-live.md` — production demo

**Production YAML (Format B — chapters/sections hierarchy):**
- Referenced in the blog post — this is what's running

**Platform GraphQL:**
- `platform/graphql-generator/.../GraphQLResolverProcessor.java`
- Annotations: `@McpDomain`, `@PlatformQuery`, `@PlatformMutation`, `@ScenarioAction`

**Existing v1 spec (Format C):**
- `parent/docs/platform/scenario-format.md`

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

Start brainstorming from scratch in a fresh session. Read the pages repo files above first, understand the three existing YAML formats and why they exist, then write a v2 spec that evolves the working system toward D1–D5.
