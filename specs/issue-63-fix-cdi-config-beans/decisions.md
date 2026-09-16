# Decisions — issue-63-fix-cdi-config-beans

## blocks#283 — Directive-Minimal Architecture

## D1: Scope — full rewrite vs incremental patching

**Choice:** Full rewrite of system prompt structure, seeding mechanism, briefing text, and deduplication removal
**Alternatives:**
- Structured data only — move goals/constraints/disposition mechanically, leave briefing text. Safe but defers the real problem.
- Briefing text + structured data — refactor prose too but keep existing rendering pipeline. Half-measure.
**Rationale:** Pre-release with a single consumer (wacky-manor). Git makes this fully reversible. Patching defers the structural issue — the directive overpowering observation sections — without solving it.
**Trade-offs:** Larger scope means more files touched across 3 repos. Mitigated by git reversibility and empirical testing.
**Sources:** blocks#283 issue description, HANDOFF.md Phase C notes
**Exploration:** quick
**Status:** captured

## D2: System prompt content — what stays in the directive

**Choice:** Name + role + voice + hard constraints. Dynamic cognitive data moves to observation sections.
**Alternatives:**
- Name + role + voice only — maximally minimal but moves safety-critical constraints to observations where LLM attention is weaker. Empirically risky.
- Full character description stays — least disruptive but keeps behavioral instructions in directive, defeating the purpose.
**Rationale:** Hard constraints (HARD severity) are safety-critical — "never reveal your true identity," "do not break character." LLM system prompts receive higher attention weight than user-turn content; moving must-not-violate rules to observation sections risks weaker adherence. Soft constraints and dynamic cognitive constraints belong in observations.
**Trade-offs:** The directive is slightly larger than pure name+role+voice. Acceptable — hard constraints are few per character and structurally distinct.
**Sources:** descriptors-composite.yaml briefing fields, SystemPromptRenderer.render()
**Exploration:** quick
**Revised by:** R1-03 (decision review round 1) — original choice was "name + role + voice only" with hard constraints treated as observations. Reviewer correctly identified that LLM attention dynamics make system prompt placement important for safety-critical constraints.
**Status:** revised

## D3: Cognitive preamble — auto-generated from active subsystems

**Choice:** Auto-generated from active subsystems. A single renderer inspects which subsystems are active and composes one coherent paragraph.
**Alternatives:**
- YAML template per archetype — define cognitive instruction templates. Readable but drifts from actual wiring.
- Inline in briefing YAML — per-character authored text. Maximum control, maximum maintenance.
- Per-subsystem snippets — each subsystem contributes its own instruction text. Modular but fragmented prose.
**Rationale:** Auto-generation eliminates drift between what's wired and what's described. Single renderer owns prose quality — reads as coherent instruction, not a list of disconnected sentences. The cognitive preamble lives in the system prompt as part of the minimal directive — it tells the agent "you have a brain, here's how to use it" without duplicating the cognitive data itself. It replaces the per-section `DirectiveSection` wrapping (which prepends behavioral instructions to each observation section). With the preamble, each observation section presents raw cognitive state; the preamble provides the meta-instruction once.
**Trade-offs:** The renderer must be updated when new subsystem types are added. Acceptable — new subsystems are infrequent and the renderer is the natural place to document their cognitive role. Losing per-section DirectiveSection instructions means each section must be self-explanatory via its heading and structure.
**Sources:** CognitionCore.promptSections(), DirectiveSection wrapping pattern
**Exploration:** quick
**Clarified by:** R1-05 (decision review round 1) — added explicit relationship to DirectiveSection and placement in system prompt.
**Status:** captured

## D4: Templates — split into voice (directive) and behavioral seeds (neurocortex)

**Choice:** Template content is split: voice/style elements (speaking patterns, catchphrases, comedic conventions) stay in the directive as part of the character's voice definition. Behavioral pattern elements (strategies, disposition biases, norms) become seed data for cognitive subsystems.
**Alternatives:**
- All template content becomes cognitive seeds — but voice/style has no natural home in CognitionCore subsystems (no "speaking style orchestrator" exists).
- Templates stay entirely in system prompt — keeps behavioral instructions in the directive, defeating the purpose.
- Remove templates entirely — loses shared archetype reuse.
**Rationale:** Templates contain two distinct categories: voice/style ("expository soliloquy," "catchphrase repetition") and behavioral patterns ("scheme obsessively," "protect allies"). Voice/style is identity (stays in directive per D2). Behavioral patterns are cognitive state (moves to seeding per D5). Clean separation along the same boundary D2 establishes.
**Trade-offs:** Templates must be manually classified into voice vs behavioral. One-time effort for 4 templates.
**Sources:** templates.yaml (cartoon-villain, cartoon-hero, protector archetypes, hanna-barbera-cartoon-style)
**Exploration:** quick
**Revised by:** R1-08 (decision review round 1) — original choice moved all template content to seeding. Reviewer identified that voice/style content has no CognitionCore subsystem home and conflicts with D2's "voice stays in directive."
**Status:** revised

## D5: Neurocortex seeding — dual-layer with existing YAML

**Choice:** Both archetype-level (template seeds) and per-character seed data, stored in existing character YAML files. A loader reads seed sections and pushes into subsystems on first boot.
**Alternatives:**
- Separate seed YAML per character — clean separation but scatters character definition across files.
- Programmatic seeding only — Java code seeds based on traits. Maximum flexibility, zero declarative visibility.
**Rationale:** Keeps all character definition in one place. Two layers compose: template provides archetype defaults, per-character overrides specialize. Both are declarative and inspectable.
**Trade-offs:** Character YAML files grow larger with seed sections. Acceptable — they're already the single source of character definition.
**Sources:** descriptors-composite.yaml, social-config.yaml, trust-evolution.yaml
**Exploration:** quick
**Status:** captured

## D6: Backward compatibility — not needed

**Choice:** No backward compatibility mechanism. Rewrite rendering directly.
**Alternatives:**
- New BriefingMode.COGNITIVE with opt-in migration — clean migration lever but unnecessary indirection.
- Replace RICH semantics — redefine existing mode. Breaks nothing since there's only one consumer.
**Rationale:** Pre-release, single consumer (wacky-manor). No external dependencies on current behavior.
**Trade-offs:** None — this is the simplest path.
**Sources:** BriefingMode enum (EMPTY/NAME_ONLY/NAME_ROLE/RICH)
**Exploration:** quick
**Status:** captured

## D7: Implementation approach — custom renderer override, not eidos modification

**Choice:** Provide a custom `SystemPromptRenderer` implementation in blocks (or wacky-manor) that overrides the eidos `@DefaultBean`. The eidos `EidosSystemPromptRenderer` stays untouched. The custom renderer produces the minimal directive (name + role + voice + hard constraints + cognitive preamble). Existing `CognitionCore.promptSections()` pipeline handles observation sections. NeurocortexSeeder (in blocks) generalizes `ManorCognitiveSeeder` to read seed data from YAML and push into subsystems on first boot.
**Alternatives:**
- Modify EidosSystemPromptRenderer directly — eidos has 113+ references to SystemPromptRenderer across eval framework, A2A card generation, and eidos-examples. Changing the eidos renderer would invalidate eval baselines and impact non-cognitive consumers.
- New CognitivePromptComposer alongside existing renderer — unnecessary indirection.
- Invert pipeline (observation sections own everything including voice/style) — too risky without empirical evidence.
**Rationale:** `@DefaultBean` override is the standard blocks pattern for specializing eidos behavior without modifying the platform. CognitionCore already does the observation-side heavy lifting. NeurocortexSeeder generalizes the existing ManorCognitiveSeeder pattern rather than introducing a parallel seeder.
**Trade-offs:** Two SystemPromptRenderer implementations exist (eidos default + cognitive override). Acceptable — the override is the intended extension point.
**Sources:** EidosSystemPromptRenderer, SystemPromptRenderer SPI, ManorCognitiveSeeder, CognitionCore.promptSections()
**Exploration:** quick
**Revised by:** R1-02 (decision review round 1) — original choice was "rewrite SystemPromptRenderer.render()" which would modify the eidos platform renderer. Reviewer identified that SystemPromptRenderer lives in eidos-api/eidos, has 113+ consumers, and should not be modified.
**Depends on:** D1 (full rewrite scope), D2 (minimal directive), D5 (seeding mechanism)
**Status:** revised

## D8: Full stack scope — no issue split

**Choice:** #283 covers the full stack: rendering rewrite + seeding mechanism + briefing text stripping + duplication removal.
**Alternatives:**
- Split rendering and seeding into separate issues — smaller PRs but intermediate state (minimal directive with empty observations) is broken and untestable.
**Rationale:** The changes are coupled — you can't test the minimal directive without seeded subsystems providing the content that was removed from the directive.
**Trade-offs:** Larger PR. Mitigated by cross-repo issue tracking (each repo gets its own issue).
**Sources:** .plan queue (6 remaining issues depend on this architectural change)
**Exploration:** quick
**Depends on:** D1 (full rewrite scope)
**Status:** captured

---

## examples#66 — Drive Adaptation from Reward Signals

## D9: Action-type to drive mapping — per-character YAML

**Choice:** Per-character mapping in social-config.yaml. Each character declares which of its drives are reinforced by which event types.
**Alternatives:**
- Global mapping — single config maps event types to drive categories. Simpler but characters lose individual response profiles; every character with a "scheming" drive is reinforced identically.
- Hybrid (global defaults + per-character overrides) — more configuration complexity for marginal benefit at this stage.
**Rationale:** Characters should respond differently to the same event type. Hooded Claw's "scheming" is reinforced by conflict_resolution; Penelope's "social-harmony" is reinforced by social_interaction. The mapping is inherently per-character because it expresses motivational identity — what this character finds rewarding.
**Trade-offs:** More YAML per character. Acceptable — social-config.yaml already defines per-character drives, goals, norms, beliefs, and relationships.
**Sources:** SocialConfig.Drive, ActionImportanceScorer event types, issue #66
**Exploration:** quick
**Status:** captured

## D10: Persistence — MindMap nodes in COGNITIVE subgraph

**Choice:** Store adapted drive intensities as properties on drive-specific MindMap nodes in the COGNITIVE subgraph.
**Alternatives:**
- Extend CaseMemoryStore — designed for episodic memories with cursor-based scanning, not mutable state documents.
- New dedicated DriveStateStore — clean interface but introduces a new persistence layer that nothing else uses.
**Rationale:** Consistent with how trust (TrustConsolidationPhase), experience (ExperienceConsolidationPhase), and other cognitive state is already stored. MindMapStore is available in both consolidation and rendering paths. Nodes support properties, confidence, and PAD values — all useful for drive state.
**Trade-offs:** MindMap nodes are a general-purpose knowledge graph mechanism, not a purpose-built drive store. Querying requires filtering by properties rather than typed queries. Acceptable — the same pattern works well for trust and experience.
**Sources:** MindMapStore, ExperienceConsolidationPhase, TrustConsolidationPhase
**Exploration:** quick
**Depends on:** D9 (per-character mapping determines what's stored per drive)
**Status:** captured

## D11: Phase design — new DriveAdaptationPhase

**Choice:** New ConsolidationPhase implementation (`DriveAdaptationPhase`) running after ExperienceConsolidationPhase (priority ~17).
**Alternatives:**
- Extend ExperienceConsolidationPhase — couples experience graduation and drive adaptation. ExperienceConsolidationPhase is in neocortex (platform), while drive adaptation needs application-level config, creating a dependency direction problem.
- Event-driven via ConsolidationCompleted CDI event — decoupled but loses access to per-tenant consolidation context and can't participate in phase ordering.
**Rationale:** Clean SRP — experience graduation and drive adaptation are separate concerns. The phase reads graduated mindmap nodes that ExperienceConsolidationPhase just created, aggregates reward signals per action-type, and updates drive intensity nodes. Runs in the existing pipeline with proper ordering.
**Trade-offs:** Second pass over data (reads nodes that ExperienceConsolidationPhase just wrote). Acceptable — the node set is small (maxPerPass=20) and the read is by subgraph, not a full scan.
**Sources:** ConsolidationPhase SPI, ConsolidationScheduler phase ordering, ExperienceConsolidationPhase
**Exploration:** quick
**Status:** captured

## D12: Reward signal — pleasure delta with arousal modulation

**Choice:** Use the memory's pleasure value as the primary reward signal. Positive pleasure = rewarding outcome (strengthens associated drive), negative = punishing (weakens it). Arousal modulates the magnitude of the adaptation effect — high-arousal experiences produce stronger updates.
**Alternatives:**
- Composite PAD reward (weighted combination of all three dimensions) — harder to reason about; high dominance is rewarding for scheming but punishing for social-harmony.
- Per-drive PAD weighting (each drive specifies which PAD dimension it responds to) — maximum fidelity but adds per-drive configuration overhead.
**Rationale:** Pleasure directly maps to "did this action produce a good outcome." It's the PAD dimension most aligned with reinforcement learning's reward concept. Arousal as a modulator captures intensity without changing direction — a calm positive experience adapts less than an exciting one. Dominance is too character-dependent to be a universal signal.
**Trade-offs:** Loses nuance from dominance dimension. Acceptable for initial implementation; per-drive PAD weighting can be added later if empirical testing shows pleasure alone is insufficient.
**Sources:** Memory.pleasure(), Memory.arousal(), issue #66, GE-20260714-439924 (multiplicative dampening)
**Exploration:** quick
**Status:** captured

## D13: Code location — reusable phase in blocks-core

**Choice:** DriveAdaptationPhase lives in blocks-core with a simple map config SPI (`Map<String, List<String>>` — event-type → drive-types). Wacky-manor provides only the YAML configuration.
**Alternatives:**
- Wacky-manor only — inherently application-specific. But the adaptation mechanics (multiplicative update, clamping, arousal modulation) are generic and reusable. Only the mapping is application-specific.
- Neocortex — alongside other consolidation phases. But creates a neocortex dependency on drive concepts which are blocks-level.
**Rationale:** The adaptation algorithm is domain-agnostic: read reward signals from graduated nodes, map to drives, apply multiplicative updates. Any application with character drives and event types can reuse it. Wacky-manor stays thin — just YAML config. Follows the blocks pattern where platform provides the engine and applications provide configuration.
**Trade-offs:** Blocks-core gains a new consolidation phase dependency on MindMapStore (neocortex). This dependency already exists via DriveOrchestrator → MemoryHygieneOrchestrator path.
**Sources:** blocks-core existing patterns (DriveOrchestrator, CognitionCore), ManorGraduationScorer, ActionImportanceScorer
**Exploration:** quick
**Depends on:** D11 (new phase), D9 (mapping SPI)
**Status:** captured

## D14: Bootstrapping — seeder writes drive nodes

**Choice:** ManorCognitiveSeeder (already enhanced in #283 for goal seeding) also seeds drive nodes into the COGNITIVE subgraph with initial intensities from social-config.yaml. DriveAdaptationPhase only reads/updates these nodes — never touches SocialConfig directly.
**Alternatives:**
- DriveConfigProvider SPI — blocks-core defines interface, wacky-manor implements to return initial intensities. Adds an SPI but avoids seeder dependency.
**Rationale:** Clean dependency direction. The seeder already runs at scenario bootstrap and handles beliefs, relationships, and goals. Adding drive seeding is natural. DriveAdaptationPhase in blocks-core has no dependency on SocialConfig — it works purely with MindMap nodes.
**Trade-offs:** Drive adaptation won't work without seeded nodes (no automatic fallback to static config). Acceptable — the seeder is mandatory for all cognitive features.
**Sources:** ManorCognitiveSeeder, D5 (neurocortex seeding from #283)
**Exploration:** quick
**Depends on:** D10 (MindMap persistence), D13 (blocks-core location)
**Status:** captured

## D15: Rendering — CharacterDrivePromptSection in blocks-core

**Choice:** Create a `CharacterDrivePromptSection` (blocks-core PromptSection) that reads adapted drive nodes from MindMapStore and renders them in the observation section pipeline. CharacterCognition stops rendering drives directly.
**Alternatives:**
- CharacterCognition reads MindMapStore — keeps rendering in wacky-manor but adds MindMapStore dependency to CharacterCognition rendering, and misaligns with #283's directive-minimal architecture.
**Rationale:** Consistent with how platform SDT drives are rendered via DrivePromptSection. Aligns with #283's architecture where all cognitive state flows through CognitionCore observation sections. CharacterCognition's drive rendering (lines 106-111) is removed — one fewer concern in the application layer.
**Trade-offs:** Two drive prompt sections exist in the observation pipeline (platform SDT DrivePromptSection + character CharacterDrivePromptSection). They render different concepts — SDT psychological needs vs character motivational identity — under separate headings. Clear enough for the LLM to distinguish.
**Sources:** DrivePromptSection, CharacterCognition.renderCognitiveSections(), D7 (CognitiveSystemPromptRenderer from #283)
**Exploration:** quick
**Depends on:** D10 (MindMap persistence), D13 (blocks-core location)
**Status:** captured
