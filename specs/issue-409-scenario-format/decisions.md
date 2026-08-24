## D1: GraphQL as primary server action type

**Choice:** GraphQL operations replace REST as the canonical server-side action in scenario YAML. Every CaseHub service must expose GraphQL — enforced at build time.
**Alternatives:**
- REST endpoints as primary delivery — loosely typed, requires separate documentation, LLMs can't discover available operations from the contract alone
- GraphQL with REST fallback — unnecessary complexity since every CaseHub service will have GraphQL
**Rationale:** GraphQL is self-describing (schema is the contract), type-safe (server validates inputs), and composable (queries select specific fields for matching). LLMs read the schema and know exactly what operations exist, what inputs they take, and what they return. The `GraphQLResolverProcessor` already generates resolvers from `@PlatformQuery`/`@PlatformMutation` on SPI interfaces, so any repo with an SPI gets GraphQL automatically.
**Trade-offs:** External platforms still integrate via REST — but REST is auto-exposed alongside GraphQL by SmallRye. The YAML format doesn't use REST; external integrators do.
**Exploration:** quick
**Status:** captured

## D2: Three action types — GraphQL, ARIA, HTTP

**Choice:** Scenario steps have three action types: GraphQL (CaseHub server operations), ARIA (frontend UI automation), HTTP (third-party REST calls). No "delivery modes" — the action type is implicit from what's being called.
**Alternatives:**
- Two types (GraphQL + ARIA only) — fails for browser-only scenarios that need to call third-party APIs
- Existing three delivery modes (rest, ui-form, simulated) — conflates delivery mechanism with action semantics; "simulated" is a runtime profile concern, not a YAML concern
**Rationale:** Clean separation of concerns. GraphQL covers all CaseHub backend operations. ARIA covers all frontend interactions using standardised accessibility vocabulary. HTTP covers external integrations and works in both backend and browser-only executors.
**Trade-offs:** Three vocabularies for an LLM to learn — but each is well-defined and standardised.
**Exploration:** quick
**Status:** captured

## D3: Direct ARIA references for frontend actions

**Choice:** Frontend steps reference ARIA roles and accessible names directly (e.g. `{ role: 'button', name: 'Submit' }`). No CSS selectors, no data-* attributes.
**Alternatives:**
- Natural language selectors resolved against ARIA tree — more readable but ambiguous, requires fuzzy matching
- CSS / data-* attribute selectors (existing spec) — coupled to implementation, not standardised, LLMs don't inherently understand them
**Rationale:** ARIA is a standardised vocabulary that LLMs already understand. Mechanically validatable against the rendered DOM. Makes scenarios accessibility-correct by design.
**Trade-offs:** Requires applications to have proper ARIA markup — but they should anyway.
**Exploration:** quick
**Status:** captured

## D4: Distributed fragment execution model

**Choice:** The backend executor partitions the YAML into fragments and distributes them to target executors (browser, other services). Each executor runs its fragment locally and autonomously — parsing the YAML, managing its own trigger graph, sequencing its own steps. Browser-only mode is the same model without a distributing backend.
**Alternatives:**
- Central command stream — backend sends one command at a time to each executor. Higher coordination overhead, single point of sequencing failure.
- Full script broadcast — every executor gets the full YAML and filters to its own steps. Wasteful, leaks information about other executors.
**Rationale:** YAML becomes a portable execution contract. Any executor (Java, browser JS, future distributed services) runs the same format. Self-contained fragments enable autonomous execution with local trigger graphs.
**Trade-offs:** Cross-executor triggers require coordination protocol between the distributing backend and fragment executors.
**Depends on:** D2 (action types determine fragment partitioning)
**Exploration:** quick
**Status:** captured

## D5: YAML submission via GraphQL

**Choice:** Scenario YAML scripts are submitted to the execution server via a GraphQL mutation. The executor is a CaseHub service with a GraphQL API like any other.
**Alternatives:**
- REST upload endpoint — inconsistent with GraphQL-first platform rule
- File-based loading — doesn't support remote/distributed submission
**Rationale:** Consistent with D1 — the executor follows the same platform rules as every other service.
**Trade-offs:** None significant — GraphQL handles string/text payloads fine.
**Depends on:** D1 (GraphQL-first platform rule)
**Exploration:** quick
**Status:** captured

## D8: Step-level target — steps are dispatch units

**Choice:** The `target` field lives on the step, not on individual commands. All commands in a step execute on one executor. Switching executors means starting a new step.
**Alternatives:**
- Command-level target — each command specifies its own executor. More flexible for interleaved browser/server sequences, but the orchestrator must decompose steps into sub-fragments per executor, which is functionally identical to having smaller steps. Creates partial failure ambiguity (browser command succeeds, server command fails — is the step failed?). Conflicts with D4's fragment model where steps are the dispatch unit.
**Rationale:** If you're switching executors, that IS a new step. The step boundary communicates the executor switch to the reader. The orchestrator sends each step to one executor wholesale — no decomposition needed. Batching consecutive same-target steps is trivial. Error model is clean: a step succeeds or fails on one executor. Command-level targets would require the orchestrator to create implicit step boundaries at every executor switch, which means the format pretends steps span executors when the runtime splits them anyway.
**Trade-offs:** Interleaved browser/server sequences produce more steps. This is a feature — each executor switch is visible in the YAML, not hidden inside a mixed step.
**Depends on:** D4 (distributed fragment execution — steps are the natural fragment boundary)
**Exploration:** quick
**Status:** captured

## D7: Explicit command objects — action as value, not key

**Choice:** Every command uses an explicit `action` field as the type discriminator: `{action: click, target: {role: button, name: Submit}}`. All three action types (ARIA, GraphQL, HTTP) share the same structure.
**Alternatives:**
- ARIA shorthand keys — action name as the YAML key (`click: {role: button, name: Submit}`). More compact for ARIA-heavy sequences, reads like prose. But only helps ARIA — GraphQL and HTTP commands are the same length either way. Creates two syntactic styles in one file. Key collision risk between action names and metadata fields. Harder to schema-validate (polymorphic keys vs uniform discriminator).
- Hybrid — allow both forms. Doubles parser complexity for style flexibility without a clear win.
**Rationale:** The format serves humans, LLMs, and parsers. Uniform structure means one pattern to learn, generate, and validate. The compactness savings from shorthand are real but marginal (one field per command) and only apply to ARIA actions while introducing inconsistency with GraphQL/HTTP commands in the same file.
**Trade-offs:** ARIA sequences are slightly more verbose than shorthand. Acceptable — the consistency across all action types outweighs per-command brevity.
**Depends on:** D2 (three action types must use the same command structure)
**Exploration:** quick
**Status:** captured

## D6: Flexible hierarchy — automations as core, scenarios as overlay

**Choice:** The base format is an automation (steps + commands, no human oversight needed). Chapters and sections are a presentation overlay for when humans pace through the execution (demos, walkthroughs). Top level can be chapters, sections, or steps directly — mutually exclusive.
**Alternatives:**
- Always full hierarchy — every automation must have chapters/sections even when running unattended, adding verbosity for no benefit
- Two-level only (steps + commands) — drops the narrative structure that makes helpdesk-style demos self-documenting and navigable
**Rationale:** Automations and scenarios are different use cases sharing the same execution engine. Simple automations (seed data, smoke tests) just need steps. Human-paced demos need narrative structure. The format shouldn't force one shape on both.
**Trade-offs:** Parser must handle three entry points (chapters, sections, steps). Worth it — the alternative is forcing demo chrome on headless automations.
**Depends on:** D4 (distributed execution model applies to both automations and scenarios)
**Exploration:** quick
**Status:** captured
