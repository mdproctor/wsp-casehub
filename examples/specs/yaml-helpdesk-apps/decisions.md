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
