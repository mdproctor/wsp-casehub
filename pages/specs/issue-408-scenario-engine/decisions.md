# Decisions — Scenario Engine Example Applications

## D1: Example applications replace standalone demo SPI impls

**Choice:** Build progressive example applications in casehub-examples instead of standalone demo SPI implementations in casehub-connectors
**Alternatives:**
- Build DemoChatPlatform + DemoCalendarPlatform as separate connector modules — solves infrastructure but has no consumer
- Build demo impls first, example apps later — supply-driven, wrong ordering
**Rationale:** The demo-spi-convention.md describes a pattern, not a build list. Demo impls only have value when an application needs them. Example apps drive demand for demo impls, not the other way around.
**Trade-offs:** No reusable shared demo modules across apps initially — each app contains its own demo alternatives. If multiple apps need the same demo impl, extract to a shared module later.
**Exploration:** quick
**Status:** captured

## D2: First example application — IT help desk (thin slice)

**Choice:** IT help desk as the first example — message in, case created, triaged, assigned, resolved, notification out
**Alternatives:**
- Order processing — classic BPM but abstract, less engaging
- Expense approval — too simple, limited connector diversity
- Pet clinic — too narrow
**Rationale:** Universally understood domain. Natural connector diversity (chat inbound, notification outbound). Exercises core platform stack: engine, work, qhorus, connectors. Scenario writes itself.
**Trade-offs:** Help desk is a well-trodden example domain — risk of feeling generic. Offset by the scenario-driven demo angle.
**Exploration:** quick
**Status:** captured

## D3: Every LLM integration point must have an SPI boundary

**Choice:** All LLM calls must sit behind an SPI with CDI alternative selection. Scripted/lookup demo impl activated by @IfBuildProfile("demo") reads pre-defined responses from scenario data.
**Alternatives:**
- Hardwire LLM calls, skip demo mode for AI steps — breaks deterministic demo capability
- Mock at test level only, not at profile level — can't run demos without LLM
**Rationale:** Scenario files must define both inputs AND expected AI outputs. Demo mode runs fully self-contained (no LLM, no external services). Live mode uses real LLM and treats scenario expectations as verification assertions.
**Trade-offs:** Every AI integration point needs an interface + CDI wiring. Small overhead per integration point but enforces clean architecture.
**Exploration:** quick
**Depends on:** D1 (example apps drive which LLM integration points need the SPI boundary first)
**Status:** captured

## D4: Demo alternatives live inside the example app, not in connector repos

**Choice:** Each example app contains its own @Alternative @IfBuildProfile("demo") implementations for the connectors and LLM providers it uses
**Alternatives:**
- Shared demo modules in connector repos (chat-demo/, calendar-demo/) — premature extraction, no second consumer yet
- Shared demo module in casehub-examples — possible later once patterns stabilise
**Rationale:** Simplest thing that works. Demo alternatives are trivial (in-memory storage, scripted responses). Extract to shared modules only when multiple apps need the same impl.
**Trade-offs:** Some duplication if two example apps use the same connector. Acceptable at this stage — duplication is cheaper than premature abstraction.
**Exploration:** quick
**Depends on:** D1
**Status:** captured

## D5: Progressive capability coverage — each slice introduces new platform capabilities

**Choice:** Example set designed as a capability coverage matrix. Each slice introduces new platform capabilities while staying small enough to understand in isolation. Capabilities may be composed/batched across slices.
**Alternatives:**
- One large reference app covering everything — overwhelming, not progressive
- Strictly one capability per slice — too many trivial examples, artificial separation
**Rationale:** End users learn progressively. Full capability coverage also means full scenario tool coverage. Composing related capabilities per slice keeps the set manageable.
**Trade-offs:** Requires upfront thought about which capabilities group well. Curriculum may evolve as the platform evolves.
**Exploration:** quick
**Depends on:** D2 (first slice establishes the pattern)
**Status:** captured

## D6: Two run modes from same scenario file

**Choice:** Each scenario file drives both demo mode (fully scripted, no external dependencies) and live mode (real connectors + LLM, scenario as verification harness)
**Alternatives:**
- Separate demo scripts and test scripts — duplication, drift
- Demo-only scenarios with separate test suites — misses the dual-purpose insight
**Rationale:** "The demo that tests itself" — a single scenario file serves as both scripted demo and automated verification. Demo mode uses pre-defined responses; live mode compares real responses against expectations.
**Trade-offs:** Scenario format must support both expected-response definitions (for demo) and verification assertions (for live). Already covered by the scenario format spec (#409).
**Exploration:** quick
**Status:** captured

## D7: Close #93, restructure epic #408 queue

**Choice:** Close #93 (Demo SPI alternatives — ChatPlatform + CalendarPlatform) as premise was wrong. Create new issue(s) under epic #408 for example applications. Reassess #94 (BankFeed/Email SPIs) — valid but independent of scenario engine epic.
**Alternatives:**
- Redefine #93 in place — confusing git/issue history
- Keep #93 open with changed scope — misleading title
**Rationale:** The scope change is fundamental. A clean new issue communicates the actual work clearly.
**Trade-offs:** Orphaned #93 in issue history. Minor — close with explanation.
**Exploration:** quick
**Depends on:** D1
**Status:** captured

---

# Distributed Executor Protocol Decisions (#418)

## D8: Orchestrator dispatches ordered step sequences to executors

**Choice:** The orchestrator groups consecutive steps bound for the same executor and dispatches them as an ordered sequence. The executor runs them in order, reports per-step progress. The orchestrator sends control messages (pause/resume/speed/step) that affect execution pace within the sequence.
**Alternatives:**
- Single-step RPC — orchestrator sends one step at a time, waits for result, decides next. Simpler but chatty (4 helpdesk steps = 4 round-trips). All sequencing in orchestrator.
- Sub-scenario with triggers — executor manages its own trigger graph and sequencing. Most autonomous but complex. The protocol should be composable enough that this can be added later (hierarchy of orchestrators) without redesigning the core.
- Progressive (start with RPC, add fragments later) — lower upfront complexity but risks protocol redesign.
**Rationale:** Ordered sequences reduce round-trips for co-located work (e.g., helpdesk: inject → verify → resolve → verify as one dispatch). Executors have enough autonomy to manage internal pacing without the complexity of local trigger evaluation. The orchestrator remains authoritative for the trigger graph and inter-executor coordination.
**Trade-offs:** Executors need lifecycle state (idle → running → paused → complete) and per-step progress reporting. More complex than single-step RPC. But the protocol should be composable — same message types at every level — so hierarchy can be layered on later.
**Sources:** ScenarioExecutor.java (current sequential executor), AriaDispatcher.java (existing single-command push protocol), scenario-handler.ts (browser executor), cross-platform scenario engine design spec §5
**Exploration:** quick
**Status:** captured
