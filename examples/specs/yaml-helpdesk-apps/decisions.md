# Decisions — YAML Helpdesk Apps

## D1: Scope — full lifecycle with eidos + LLM, not just CRUD

**Choice:** Full case lifecycle with eidos agent descriptors and LLM-based classification, expressed in YAML. Not a port of the Java version — an upgrade that adds agentic capabilities the Java version doesn't have.
**Alternatives:**
- CRUD + rich playbooks — playbooks orchestrate workflow via step sequences, loses reactive bindings
- CRUD only — brings Pages helpdesk as-is, doesn't demonstrate case engine
**Rationale:** The goal is to prove that a complete case-managed application requires zero Java. CRUD alone doesn't demonstrate the case engine's value.
**Trade-offs:** Requires platform foundation work (auto-discovery, SPI adapter)
**Sources:** Existing Pages helpdesk app.yaml, existing Java helpdesk case definition
**Exploration:** quick
**Status:** captured

## D2: Data integration — SPI adapter, not a bridge

**Choice:** Standard DataProvider SPI implementation backed by JSON collections. Case instances are JSON documents in the collection store. Uniform data access.
**Alternatives:**
- Special bridge/sync layer between case engine and JSON collections — two data models with sync
- Case engine owns data, JSON collections are a read-only view — projection sync, stale reads
**Rationale:** User direction: "this is surely just about an SPI to user data? all user data should be abstracted with uniform access. this isn't a special case." The case engine shouldn't know what backs it.
**Trade-offs:** None significant — this is the clean architecture
**Sources:** CaseInstanceCache SPI, JsonCollectionStore, DataProvider SPI
**Exploration:** quick
**Status:** captured

## D3: Event paradigm — push topics as CDI equivalent

**Choice:** Push topics serve as the event bus. Workers publish to topics from YAML `do:` blocks. Same pub/sub paradigm as CDI but over WebSocket transport, declarable in YAML.
**Alternatives:**
- REST-only — workers call HTTP endpoints, no event bus
- Full CDI port to browser — over-engineered, wrong abstraction level
**Rationale:** Pages push infrastructure already exists with topics and WebSocket delivery. Workers' `do:` blocks can `call: http` to a publish endpoint. The paradigm (pub/sub with topics) maps naturally to both YAML declarations and TS decorators.
**Trade-offs:** Needs a REST publish endpoint on the push infrastructure
**Sources:** casehub-pages-push, HelpdeskPushEndpoint, TicketPushObserver
**Exploration:** deep-analysis
**Status:** captured

## D4: Two authoring modes — YAML and TS decorators

**Choice:** Support both YAML bindings and TS decorators (`@Observes`, `@Publish`) as first-class authoring modes. Same runtime underneath.
**Alternatives:**
- YAML only — simpler but limits web developer appeal
- TS only — loses the LLM-generation and visual review story
**Rationale:** Two audiences. LLM-generated apps need YAML (parseable, visualisable). Web developers need TS decorators (familiar, IDE-supported). Both compile to the same paradigm: bindings + workers + push topics.
**Trade-offs:** More platform surface area to build and maintain
**Sources:** Lit decorator patterns (@customElement, @property), CDI @Observes pattern
**Exploration:** deep-analysis
**Status:** captured
**Depends on:** D3 (push topics as the shared event bus)

## D5: Playbook visualiser as LLM review surface

**Choice:** Case YAML bindings and `do:` blocks must be structured so the playbook visualiser can render them as steps in a visual flow. The YAML is simultaneously the runtime configuration and the visual model.
**Alternatives:**
- Separate visual model — maintain a separate representation for the visualiser
- Text-based review only — reviewers read YAML directly
**Rationale:** User direction: "my hope is that these playbook UIs are easier for reviewing what an LLM generates as a whole." Visual review scales; reading YAML doesn't. Making the case YAML renderable in the visualiser means LLM-generated apps get automatic visual review.
**Trade-offs:** Constrains the YAML schema design — must be visualiser-friendly
**Sources:** Pages playbook visualiser, helpdesk-demo.yaml playbook
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (full lifecycle in YAML)

## D6: Side-by-side comparison

**Choice:** New `helpdesk-apps/` module alongside existing `helpdesk/`. Both remain active, unchanged.
**Alternatives:**
- Replace helpdesk/ with the YAML version — loses the comparison
- Subdirectory within helpdesk/ — muddies the existing module
**Rationale:** The comparison tells the story: 20 Java classes vs 5 YAML files, same behaviour. Both buildable, both runnable.
**Trade-offs:** Two modules to maintain
**Sources:** Existing helpdesk/ directory structure
**Exploration:** quick
**Status:** captured

## D7: Eidos + LLM as first-class showcase

**Choice:** The YAML helpdesk formalizes the agentic patterns the Java version already demonstrates (workers-as-agents, capability dispatch, dynamic routing) using eidos descriptors, and upgrades the classifier from keyword matching to LLM-based via `call: agent` + agent-config.
**Alternatives:**
- Keep keyword matching in YAML (switch: block) — preserves Java version's approach but doesn't showcase eidos/LLM
- Add eidos only without LLM — demonstrates agent identity but not the classification upgrade
**Rationale:** The Java helpdesk already has the agentic pattern — workers with capabilities, capability-based dispatch, dynamic routing. Eidos formalizes those identities. LLM classification demonstrates the natural upgrade path from keyword matching to semantic understanding. Same pattern, better implementation.
**Trade-offs:** Requires API key for LLM provider; non-deterministic classification
**Sources:** casehub-eidos (AgentDescriptor, AgentRegistry, SystemPromptRenderer), casehub-platform-agent-config-core (AgentManifest), wacky-manor eidos usage patterns
**Exploration:** deep-analysis
**Status:** captured
**Depends on:** D1 (full lifecycle scope)

## D8: LLM / simulator dual-wiring — every demo runs both ways

**Choice:** The agent manifest wires agents to either a real LLM provider or a simulator. Switchable via config property or Quarkus profile. Simulator is the default for demos — no API key needed, deterministic responses. LLM mode available for live demos showing semantic classification.
**Alternatives:**
- LLM only — requires API key for every demo run, non-deterministic
- Simulator only — doesn't demonstrate real LLM capability
- Separate demo apps for each mode — duplicates the example
**Rationale:** The helpdesk was built before platform simulation existed. Now every demo should demonstrate both modes. The simulator config also serves as a teaching tool — users see how simulation rules map to LLM behaviour. This pattern should be standard for all examples.
**Trade-offs:** Simulator rules must be kept in sync with what the LLM would actually produce
**Sources:** casehub-platform-simulation-config, casehub-platform-agent-simulation-core, casehub-platform-simulation-testing
**Exploration:** quick
**Status:** captured
**Depends on:** D7 (eidos + LLM)

## D9: Playbooks as dual-purpose — teaching AND integration tests

**Choice:** Playbook YAML serves both as interactive teaching walkthrough (browser) and as headless integration tests (Playwright/CI). `verify:` steps with expected values become pass/fail assertions in headless mode. Same artifact, two execution modes.
**Alternatives:**
- Separate test suite — Java/TS test classes alongside playbooks, duplicating the scenario logic
- Playbooks for teaching only — separate integration tests in a different format
**Rationale:** The playbook already has verification steps (verify-ticket-exists, verify-notification). Formalizing assertions makes the playbook a testable artifact. For LLM-generated apps, this closes the loop: generate YAML → review visually → test headlessly. No separate test harness.
**Trade-offs:** Headless runner needs to handle timing (async bindings, eventual consistency). Verify steps need retry/poll semantics for reactive flows.
**Sources:** helpdesk-demo.yaml verification steps, Pages playbook infrastructure, Playwright headless execution
**Exploration:** quick
**Status:** captured
**Depends on:** D5 (playbook visualiser), D8 (simulator for deterministic test runs)

## D10: Three-way viewer — YAML / visual / split

**Choice:** All YAML artifacts (case definitions, playbooks, eidos descriptors, agent manifest) viewable in three modes: raw YAML, graphical visualisation, or split view with linked navigation. Clicking a visual step highlights the corresponding YAML; editing YAML live-updates the visual.
**Alternatives:**
- YAML only (current) — doesn't leverage the graphical viewer
- Visual only — loses the ability to see/edit the source
- Two separate views without linking — no navigation between them
**Rationale:** The split view makes the relationship between declaration and behaviour explicit. Essential for learning, LLM review, and debugging. The graphical playbook viewer already exists — this is about surfacing it alongside the YAML in a unified experience.
**Trade-offs:** Live YAML→visual update needs incremental parsing; malformed YAML mid-edit must degrade gracefully
**Sources:** Pages playbook visualiser, existing YAML viewer in helpdesk demo
**Exploration:** quick
**Status:** captured
**Depends on:** D5 (playbook visualiser)
