# Decisions — blocks#283 Directive-Minimal Architecture

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
