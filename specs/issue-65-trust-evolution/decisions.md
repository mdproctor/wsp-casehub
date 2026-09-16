## D1: Trust computation engine — Ledger's TrustScoreComputer with in-memory persistence

**Choice:** Use the ledger's Bayesian Beta trust model (`TrustScoreComputer`) with in-memory ledger stores (`casehub-ledger-memory`) rather than building a custom trust computation.
**Alternatives:**
- Custom event-counting heuristic in wacky-manor — simpler but duplicates what ledger already provides, misses temporal decay and credibility weighting
- Full persistent ledger pipeline with JPA — unnecessary for a game session; in-memory stores are designed for exactly this use case
**Rationale:** TrustScoreComputer is pure Java with no CDI dependency, designed for standalone use. The in-memory stores are already in wacky-manor's dependency tree. This makes wacky-manor an exemplar of lightweight ledger usage — real Bayesian Beta scoring without database overhead.
**Trade-offs:** Ledger entry/attestation objects are heavier than a simple counter, but the computation quality (temporal decay, credibility, Bayesian posterior) justifies the structure.
**Sources:** io.casehub.ledger.core.trust.TrustScoreComputer, io.casehub.ledger.memory.InMemoryLedgerEntryRepository, GE-20260429-42fb02 (Bayesian Beta trust returns 0.5 for no evidence)
**Exploration:** quick
**Status:** captured

## D2: Boundary — reusable framework in blocks, YAML-configurable

**Choice:** All trust evolution framework classes go in blocks-core. Trust event classification, scoring parameters, and thresholds are YAML-configurable via a `TrustEvolutionConfigSpec` in `blocks/agentic-yaml/spec/cognition/`. Wacky-manor writes zero trust-specific Java — only YAML configuration.
**Alternatives:**
- All in wacky-manor — not reusable; next cognitive agent rebuilds the same pipeline
- Split: framework in blocks, app-specific recorder in wacky-manor — the recorder logic (create entry + attestation for action type) is generic when parameterised by config; no app-specific Java needed
**Rationale:** The action→verdict mapping is a configuration table, not application logic. blocks already has cognition spec types (DriveConfigSpec, RetentionConfigSpec, etc.) — trust fits the same pattern. Any cognitive agent gets trust evolution by adding YAML config.
**Trade-offs:** blocks-core gains new dependencies on casehub-neocortex-mindmap-api and casehub-ledger-core. Acceptable — blocks is already the integration layer between ledger and neocortex.
**Sources:** blocks/agentic-yaml/spec/cognition/ (existing cognitive spec pattern), DSL Style Guide §YAML/Java Parity Principle
**Exploration:** deep-analysis
**Status:** captured

## D3: Per-relationship trust via attestor-filtered scoring

**Choice:** Each observer creates attestations (with their agentId as attestorId) on the target's ledger entries. During consolidation, TrustScoreComputer is called with attestations filtered to a single attestorId, producing per-relationship Bayesian Beta scores.
**Alternatives:**
- Capability-tag namespacing (e.g. "relationship:penelope") — works but overloads the capability concept for something that's really attestor identity
- Global per-actor trust (no per-relationship) — loses the core insight: Penelope trusts HC differently than Muttley trusts HC
**Rationale:** TrustScoreComputer already accepts filtered attestation lists. Filtering by attestorId is the natural way to partition trust — it's what attestorId semantically means.
**Trade-offs:** Requires N×M scoring calls during consolidation (N observers × M targets). Acceptable for game-scale character counts (< 20).
**Sources:** io.casehub.ledger.core.trust.TrustScoreComputer#compute, io.casehub.ledger.api.model.LedgerAttestation#attestorId
**Exploration:** quick
**Status:** captured

## D4: Wacky-manor YAML integration — hand-rolled loader for now, DSL migration follow-up

**Choice:** For #65, add trust config to social-config.yaml using the existing ManorSocialConfigLoader pattern. The blocks-side TrustEvolutionConfigSpec is built (for DSL consumers), but wacky-manor doesn't use the agentic-yaml pipeline yet. A follow-up issue ports all of wacky-manor's social config to the blocks YAML DSL.
**Alternatives:**
- Bundle the full DSL migration into #65 — correct end state but doubles the scope; the migration touches drives, norms, beliefs, relationships, orchestrator wiring
- Skip the blocks spec type, only build for hand-rolled consumption — loses the DSL-first design; the spec type costs nothing to build alongside the framework classes
**Rationale:** Wacky-manor's config pipeline predates the blocks YAML work. The port is valuable (proves the DSL works end-to-end for cognitive agents) but orthogonal to trust evolution. Building the spec type now means the follow-up migration picks it up automatically.
**Trade-offs:** Short-term duplication — ManorSocialConfigLoader parses trust config into a record that parallels TrustEvolutionConfigSpec. Eliminated when the DSL migration lands.
**Sources:** io.casehub.examples.manor.agent.ManorSocialConfigLoader, social-config.yaml
**Exploration:** deep-analysis
**Status:** captured

## D5: Overlay property model — trust-score + alpha/beta posterior

**Choice:** Write three properties on person-entity overlay nodes during consolidation: `trust-score` (Bayesian Beta mean, 0.0-1.0), `trust-alpha`, and `trust-beta` (posterior parameters). The posterior parameters allow subsequent consolidation cycles to continue the Bayesian update without recomputing from the full attestation history.
**Alternatives:**
- trust-score only — loses the posterior; every consolidation cycle must recompute from all attestations
- Encode trust in PAD dimensions (e.g. dominance = trust) — conflates two different concepts; PAD is affective perception, trust is epistemic
**Rationale:** Carrying alpha/beta forward is O(1) per cycle vs O(N) for full recomputation. The properties are cheap (string key-value on existing overlay nodes).
**Trade-offs:** Three properties per overlay node instead of one. Negligible cost.
**Sources:** io.casehub.neocortex.mindmap.NodeUpdate#withPropertiesToSet, io.casehub.ledger.core.trust.TrustScoreComputer.ActorScore (alpha, beta fields)
**Exploration:** quick
**Status:** captured

## D6: Consolidation ordering — @Priority(16), after experience graduation

**Choice:** TrustConsolidationPhase runs at @Priority(16), immediately after ExperienceConsolidationPhase (15).
**Alternatives:**
- Same priority (15) — non-deterministic ordering between phases
- Much later (e.g. 50) — unnecessary gap; trust scoring logically follows experience graduation
**Rationale:** Trust scoring reads from ledger entries (not episodic buffer), so it doesn't strictly depend on experience graduation. But running after it keeps the ordering intuitive: events graduate, then trust is computed.
**Trade-offs:** None significant.
**Sources:** io.casehub.neocortex.mindmap.intelligence.consolidation.ExperienceConsolidationPhase (@Priority(15))
**Exploration:** quick
**Status:** captured

## D7: Trust event recording — CDI event observer on action completion

**Choice:** The TrustEventRecorder in blocks observes a CDI event fired on action completion. It maps the action type to a verdict using the YAML-configured trust event table, creates a LedgerEntry for the acting character, and creates LedgerAttestations for affected and witnessing characters.
**Alternatives:**
- Direct call in the tick loop — couples the orchestrator to trust recording; every cognitive agent would need to add the call manually
- Post-action interceptor/decorator — more complex wiring for the same effect
**Rationale:** CDI observer is the established blocks pattern for cross-cutting concerns. The orchestrator fires the event; the recorder is a zero-touch add-on activated by YAML config presence.
**Trade-offs:** Requires a CDI action-completed event to exist (may need to be added to the orchestrator if not present).
**Sources:** blocks CDI observer patterns (existing), ScenarioOrchestrator tick loop
**Exploration:** quick
**Status:** captured

## D8: Observation threshold — configurable delta gating

**Choice:** Trust changes only surface in observations when the absolute delta since the last rendered trust observation exceeds a configurable threshold (default 0.15). Below that, the score updates on the overlay node but no observation text is generated.
**Alternatives:**
- Always render — noisy; small fluctuations produce meaningless observations
- Fixed threshold levels only (HIGH/MODERATE/LOW transitions) — loses granularity; a 0.69→0.71 transition across the MODERATE→HIGH boundary fires but a 0.3→0.5 change within MODERATE doesn't
**Rationale:** Delta-based gating captures significant trust shifts regardless of which level band they fall in. Combined with TrustLevel thresholds for the observation text ("You've come to rely on..." vs "Something makes you uneasy..."), this gives both sensitivity and meaningful rendering.
**Trade-offs:** Requires tracking "last rendered trust score" per relationship — one additional property on the overlay node (`trust-last-rendered`).
**Sources:** io.casehub.examples.manor.agent.PerceptionTranslator (existing distance threshold pattern at 0.4)
**Exploration:** quick
**Status:** captured
