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
**Preamble wording constraint:** The preamble describes cognitive architecture ("You have motivational drives that influence your priorities"), NOT current state ("some are stronger than others right now"). State-dependent claims conflict with the system prompt caching strategy — the preamble is part of the cached RenderedPrompt, and state-dependent text would require per-turn cache invalidation. Actual cognitive state is conveyed by observation sections, which are rebuilt every turn. The preamble should also disambiguate the two drive systems (see D16): "Your psychological needs — curiosity, competence, affiliation, autonomy — shift based on your interactions. Separately, your character motivations — the drives that define who you are — evolve based on your experiences."
**Sources:** CognitionCore.promptSections(), DirectiveSection wrapping pattern, RenderedPromptCache
**Exploration:** quick
**Clarified by:** R1-05 (decision review round 1) — added explicit relationship to DirectiveSection and placement in system prompt. R1-06 + R1-07 (decision review round 2) — preamble must describe architecture not current state (caching constraint); added drive disambiguation text for D16.
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
**Rationale:** `EidosSystemPromptRenderer` is `@ApplicationScoped` — not `@DefaultBean`. Overriding it requires `@Alternative @Priority(1)` on the replacement bean (a standard CDI mechanism). A `@DefaultBean` would cause an ambiguous dependency error since the existing bean is `@ApplicationScoped` and not automatically displaced. The `@Alternative @Priority(1)` approach is the correct CDI pattern for overriding a platform `@ApplicationScoped` bean without modifying the platform. CognitionCore already does the observation-side heavy lifting. NeurocortexSeeder generalizes the existing ManorCognitiveSeeder pattern rather than introducing a parallel seeder.
**Trade-offs:** Two SystemPromptRenderer implementations exist (eidos default + cognitive override). Acceptable — `@Alternative` with explicit priority makes the override intentional and priority-ordered.
**Sources:** EidosSystemPromptRenderer (`@ApplicationScoped`), SystemPromptRenderer SPI, ManorCognitiveSeeder, CognitionCore.promptSections()
**Exploration:** quick
**Revised by:** R1-02 (decision review round 1) — original choice was "rewrite SystemPromptRenderer.render()" which would modify the eidos platform renderer. Reviewer identified that SystemPromptRenderer lives in eidos-api/eidos, has 113+ consumers, and should not be modified. Round 2 (R1-02): corrected rationale — replaced incorrect `@DefaultBean` language with the actual CDI mechanism (`@Alternative @Priority(1)` overriding `@ApplicationScoped`).
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
- Separate drive-specific subgraph (e.g., `drives-{agentId}`) — cleaner separation but proliferates subgraphs; ExperienceConsolidationPhase already uses a single `cognitive` type subgraph for all agents.
**Rationale:** Consistent with how trust (TrustConsolidationPhase), experience (ExperienceConsolidationPhase), and other cognitive state is already stored. MindMapStore is available in both consolidation and rendering paths. Nodes support properties, confidence, and PAD values — all useful for drive state.
**Node discrimination:** Drive nodes share the `cognitive` type subgraph used by ExperienceConsolidationPhase. They are distinguished from graduated experience nodes by: (1) `cognitiveKind: "drive-intensity"` (experience nodes use classifier-assigned `cognitiveKind` from `GraduationResult`); (2) `provenance: "drive-adaptation"` (experience nodes use `provenance: "experience-consolidation"`). Drive nodes carry: `agent-id` (agent scoping, same pattern as experience nodes), `drive-type` (e.g., "scheming", "social-harmony"), `intensity` (current adapted value, 0.0–1.0), `initial-intensity` (from seed data). DriveAdaptationPhase queries by `cognitiveKind: "drive-intensity"` + `agent-id` to locate drive nodes.
**Trade-offs:** MindMap nodes are a general-purpose knowledge graph mechanism, not a purpose-built drive store. Querying requires filtering by properties rather than typed queries. Acceptable — the same pattern works well for trust and experience.
**Sources:** MindMapStore, ExperienceConsolidationPhase (cognitiveKind property, agent-id property), TrustConsolidationPhase
**Exploration:** quick
**Clarified by:** R1-08 (decision review round 2) — added subgraph selection, node discrimination mechanism, and property model for drive nodes.
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

## D12: Reward signal — configurable PAD axis with arousal modulation

**Choice:** Each drive reinforcement mapping specifies which PAD dimension serves as its reward signal (default: pleasure). Arousal modulates the magnitude. The reward axis is a per-mapping configuration — not per-drive, not global — because the same drive may respond to different PAD dimensions depending on the event type.
**Alternatives:**
- Pleasure-only (original choice) — simple but mismodels outcomes where instrumental success diverges from hedonic experience (e.g., successful but stressful conflict mediation weakens social-harmony under pleasure-only reward).
- Composite PAD reward (weighted combination of all three dimensions) — harder to reason about and configure.
- Per-drive PAD weighting (each drive specifies a single PAD dimension globally) — ignores that the same drive may need different signals for different event types.
**Rationale:** Pleasure is the right reward axis for most mappings — it captures "was this a good outcome." But for drives like social-harmony, dominance (agency/control in the situation) is a better reward signal for conflict-resolution events: a stressful mediation that succeeds is high-dominance, not high-pleasure. Making the axis configurable per mapping is consistent with D9's per-character philosophy — the reinforcement relationship is already per-character, extending it to include the reward dimension is architecturally coherent. Default to pleasure when not specified.
**Trade-offs:** Adds one optional field to each mapping entry. Marginal configuration cost for significantly better modeling of the pleasure/instrumental-success divergence.
**Sources:** Memory.pleasure(), Memory.arousal(), Memory.dominance(), issue #66, GE-20260714-439924 (multiplicative dampening)
**Exploration:** quick
**Revised by:** R1-03 (decision review round 2) — original pleasure-only choice mismodeled successful-but-unpleasant outcomes. Reviewer demonstrated concrete failure scenario (Penelope's social-harmony weakened by stressful conflict mediation). Per-mapping PAD axis selection is consistent with D9's per-character philosophy.
**Status:** revised

## D13: Code location — reusable phase in blocks-core

**Choice:** DriveAdaptationPhase lives in blocks-core with a structured config SPI: `Map<String, List<DriveReinforcementEntry>>` (event-type → list of reinforcement entries). Each `DriveReinforcementEntry` specifies: `driveType` (string), `direction` (POSITIVE or NEGATIVE, default POSITIVE), and `rewardAxis` (PLEASURE, DOMINANCE, or COMPOSITE, default PLEASURE). Wacky-manor provides only the YAML configuration.
**Alternatives:**
- Simple map `Map<String, List<String>>` (original choice) — lacks reinforcement direction and reward axis configuration. Cannot express negative reinforcement (e.g., socializing weakens scheming) or per-mapping PAD axis selection (per D12 revision).
- Wacky-manor only — inherently application-specific. But the adaptation mechanics (multiplicative update, clamping, arousal modulation) are generic and reusable.
- Neocortex — alongside other consolidation phases. Creates a neocortex dependency on drive concepts which are blocks-level.
**Rationale:** The adaptation algorithm is domain-agnostic: read reward signals from graduated nodes, map to drives using the structured SPI, apply directional multiplicative updates. The enriched SPI enables: (1) negative reinforcement — `social_interaction → scheming: NEGATIVE` means pleasant socializing weakens scheming identity; (2) per-mapping reward axis — `conflict_resolution → social-harmony: POSITIVE, rewardAxis=DOMINANCE` uses dominance instead of pleasure as the reward signal (per D12). Follows the blocks pattern where platform provides the engine and applications provide configuration.
**Trade-offs:** Richer SPI requires `DriveReinforcementEntry` record instead of plain strings. Marginal complexity for significantly better expressiveness. Blocks-core gains a new consolidation phase dependency on MindMapStore (neocortex). This dependency already exists via DriveOrchestrator → MemoryHygieneOrchestrator path.
**Sources:** blocks-core existing patterns (DriveOrchestrator, CognitionCore), ManorGraduationScorer, ActionImportanceScorer
**Exploration:** quick
**Revised by:** R1-04 (decision review round 2) — original `Map<String, List<String>>` SPI lacked reinforcement direction and reward axis selection. Enriched SPI is consistent with D9's per-character philosophy and D12's revised reward signal approach.
**Depends on:** D11 (new phase), D9 (mapping SPI), D12 (reward signal — per-mapping axis)
**Status:** revised

## D14: Bootstrapping — seeder writes drive nodes

**Choice:** ManorCognitiveSeeder (already enhanced in #283 for goal seeding) also seeds drive nodes into the COGNITIVE subgraph with initial intensities from social-config.yaml. DriveAdaptationPhase only reads/updates these nodes — never touches SocialConfig directly.
**Alternatives:**
- DriveConfigProvider SPI — blocks-core defines interface, wacky-manor implements to return initial intensities. Adds an SPI but avoids seeder dependency.
**Rationale:** Clean dependency direction. The seeder already runs at scenario bootstrap and handles beliefs, relationships, and goals. Adding drive seeding is natural. DriveAdaptationPhase in blocks-core has no dependency on SocialConfig — it works purely with MindMap nodes.
**Trade-offs:** Drive adaptation won't work without seeded nodes (no automatic fallback to static config). Acceptable — the seeder is mandatory for all cognitive features.
**Diagnostic requirement:** DriveAdaptationPhase must emit a `WARNING`-level log message when it finds no drive nodes for a given `agent-id` in the COGNITIVE subgraph. Unlike experience nodes (whose absence is expected at cold start), drive nodes are required at bootstrap — their absence is a configuration error. The diagnostic should log the agent-id and tenant-id to aid troubleshooting.
**Sources:** ManorCognitiveSeeder, D5 (neurocortex seeding from #283)
**Exploration:** quick
**Clarified by:** R1-11 (decision review round 2) — added diagnostic logging requirement for missing drive nodes.
**Depends on:** D10 (MindMap persistence), D13 (blocks-core location)
**Status:** captured

## D15: Rendering — CharacterDrivePromptSection in blocks-core

**Choice:** Create a `CharacterDrivePromptSection` (blocks-core PromptSection) that reads adapted drive nodes from MindMapStore and renders them in the observation section pipeline. CharacterCognition stops rendering drives directly.
**Alternatives:**
- CharacterCognition reads MindMapStore — keeps rendering in wacky-manor but adds MindMapStore dependency to CharacterCognition rendering, and misaligns with #283's directive-minimal architecture.
**Rationale:** Consistent with how platform SDT drives are rendered via DrivePromptSection. Aligns with #283's architecture where all cognitive state flows through CognitionCore observation sections. CharacterCognition's drive rendering (lines 106-111) is removed — one fewer concern in the application layer.
**Trade-offs:** Two drive prompt sections exist in the observation pipeline (platform SDT DrivePromptSection + character CharacterDrivePromptSection). They render different concepts — SDT psychological needs vs character motivational identity — under separate headings with explicit differentiation.
**Heading differentiation:** DrivePromptSection renders under "Psychological Needs" (via `CognitiveObservationSections.motivationalStateSection()`). CharacterDrivePromptSection renders under "Character Motivations". The cognitive preamble (D3) provides disambiguation text explaining the relationship between the two systems (see D16).
**Sources:** DrivePromptSection, CharacterCognition.renderCognitiveSections(), D7 (CognitiveSystemPromptRenderer from #283)
**Exploration:** quick
**Clarified by:** R1-07 (decision review round 2) — added explicit heading differentiation and preamble disambiguation requirement.
**Depends on:** D10 (MindMap persistence), D13 (blocks-core location), D16 (drive taxonomy relationship)
**Status:** captured

---

## Cross-Cutting Implicit Decisions (surfaced by decision review round 2)

## D16: Drive taxonomy — SDT drives and character motivations are intentionally independent

**Choice:** SDT drives (DriveOrchestrator, 4 fixed axes: CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY) and character motivations (SocialConfig.Drive, free-form: "scheming", "gallantry", "social-harmony") remain independent systems. They are rendered separately, adapted by separate mechanisms, and stored in separate locations. The cognitive preamble (D3) provides disambiguation text.
**Alternatives:**
- Convergence: character motivations as specializations of SDT axes — e.g., "scheming" maps to AUTONOMY. Loses the modeling distinction (SDT axes are universal psychological needs; character motivations are authored identity).
- Subsumption: one system replaces the other. SDT drives are platform-level and dynamically computed; character motivations are application-level and identity-defining. Neither subsumes the other's function.
- Explicit divergence with no disambiguation — current implicit state. The LLM receives two drive sections with no framing of their relationship.
**Rationale:** SDT drives and character motivations model motivation at different abstraction levels. SDT drives capture universal psychological needs (computed dynamically from interaction patterns by `DriveOrchestrator`). Character motivations capture character-specific motivational identity (authored per character, adapted from reward signals). They are complementary, not competing: SDT AFFILIATION being high and `social-harmony` being high are reinforcing signals at different levels of description. The preamble disambiguates this for the LLM.
**Trade-offs:** The LLM must interpret two drive systems. Mitigated by distinct section headings (D15) and preamble disambiguation (D3).
**Sources:** DriveOrchestrator (4-axis SDT model), SocialConfig.Drive (free-form character motivations), DrivePromptSection, CharacterDrivePromptSection
**Exploration:** quick (surfaced by R1-14, decision review round 2)
**Status:** captured

## D17: CognitionConfig CDI scope — application-level, not per-agent

**Choice:** `CognitionConfig` registered as a CDI synthetic bean via `SyntheticBeanBuildItem` with `@DefaultBean` scope. One config per application. Per-agent cognitive variation is expressed through the data fed to subsystems (SocialConfig), not through CognitionConfig flags.
**Alternatives:**
- Per-agent CognitionConfig (e.g., via `@RequestScoped` or a provider pattern) — would allow agents with different subsystem configurations in the same application. Adds significant CDI complexity for a capability not currently needed.
- No CDI registration (keep as plain record) — CognitiveSystemPromptRenderer can't inject it; must receive it through a different mechanism.
**Rationale:** All cognitive subsystem orchestrators (MoodOrchestrator, DriveOrchestrator, etc.) are `@ApplicationScoped` CDI beans — the same subsystem instances serve all agents. CognitionConfig flags describe which of these application-scoped subsystems are active. Per-agent variation in what subsystems exist would require per-agent CDI scoping of the subsystems themselves — a fundamentally different architecture. The CognitionConfig synthetic bean describes the application's cognitive architecture, matching the CDI scope of the subsystems it configures. If a future application needs agents with different cognitive subsystem configurations, that is a different deployment, not a per-agent CognitionConfig.
**Known limitation:** All agents in a given application share the same CognitionConfig. This is architecturally correct for the current design where subsystems are application-scoped.
**Sources:** CognitionConfig (plain record), SocialAvatarCognition (creates one CognitionCore for all agents), CognitionConfigBuildItem, AgenticYamlProcessor.discoverCognitionConfig()
**Exploration:** quick (surfaced by R1-09, decision review round 2)
**Status:** captured

## D18: CharacterCognition deduplication — remove CognitionCore sections

**Choice:** `CharacterCognition.renderCognitiveSections()` stops calling `cognitionCore.promptSections()`. CognitionCore sections are wired into the observation pipeline at the `ObservationBuilder` level, separate from CharacterCognition's application-specific sections.
**Alternatives:**
- Keep CognitionCore call in CharacterCognition — simplest but creates a second path for CognitionCore sections alongside SocialAvatarCognition's `buildSections()`. In rendering paths where both contribute, CognitionCore sections would be duplicated.
- Remove from both CharacterCognition and SocialAvatarCognition, wire at a higher level — cleanest but requires both paths to be refactored simultaneously.
**Rationale:** The spec's "After" architecture shows CognitionCore sections and CharacterCognition sections as independent items in the observation pipeline. CharacterCognition should provide only application-specific content that has no CognitionCore equivalent: character motivations, initial beliefs, trust perceptions, social awareness, norms. CognitionCore sections (mood, SDT drives, narrative, mental model, user model, strategy, goals, personality, soft constraints) are provided by CognitionCore through the observation pipeline wiring, not aggregated by CharacterCognition. In the ScenarioOrchestrator path, ObservationBuilder receives CognitionCore sections as a separate input alongside CharacterCognition sections.
**Sources:** CharacterCognition.renderCognitiveSections() (lines 141-148), SocialAvatarCognition.buildSections(), spec §5 (deduplication)
**Exploration:** quick (surfaced by R1-15, decision review round 2)
**Depends on:** D7 (CognitiveSystemPromptRenderer), D15 (CharacterDrivePromptSection)
**Status:** captured
