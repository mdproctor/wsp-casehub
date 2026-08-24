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
