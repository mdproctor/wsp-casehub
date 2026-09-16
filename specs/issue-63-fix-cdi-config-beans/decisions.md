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

**Choice:** Name + role + voice only. Everything behavioral moves to observation sections.
**Alternatives:**
- Name + role + voice + hard constraints — keep safety-critical constraints in directive. Adds conditional logic for constraint severity.
- Full character description stays — least disruptive but keeps behavioral instructions in directive, defeating the purpose.
**Rationale:** Minimal directive surface maximizes the influence of dynamic cognitive sections. Hard constraints are not special — they're just high-priority observations.
**Trade-offs:** Voice/style instructions in the system prompt may still influence behavior more than we want. Empirical testing needed.
**Sources:** descriptors-composite.yaml briefing fields, SystemPromptRenderer.render()
**Exploration:** quick
**Status:** captured

## D3: Cognitive preamble — auto-generated from active subsystems

**Choice:** Auto-generated from active subsystems. A single renderer inspects which subsystems are active and composes one coherent paragraph.
**Alternatives:**
- YAML template per archetype — define cognitive instruction templates. Readable but drifts from actual wiring.
- Inline in briefing YAML — per-character authored text. Maximum control, maximum maintenance.
- Per-subsystem snippets — each subsystem contributes its own instruction text. Modular but fragmented prose.
**Rationale:** Auto-generation eliminates drift between what's wired and what's described. Single renderer owns prose quality — reads as coherent instruction, not a list of disconnected sentences.
**Trade-offs:** The renderer must be updated when new subsystem types are added. Acceptable — new subsystems are infrequent and the renderer is the natural place to document their cognitive role.
**Sources:** CognitionCore.promptSections(), DirectiveSection wrapping pattern
**Exploration:** quick
**Status:** captured

## D4: Templates — become cognitive seed defaults

**Choice:** Template content becomes seed data for cognitive subsystems (initial strategies, disposition biases, norms). Templates move from "prompt text injection" to "neurocortex seeding."
**Alternatives:**
- Templates stay in system prompt — treat as structural style guides. Keeps behavioral instructions in directive.
- Remove templates entirely — per-character seed data replaces them. Loses shared archetype reuse.
**Rationale:** Templates are behavioral patterns (villain archetype, hero archetype). These are exactly the kind of cognitive defaults that subsystems should own. Shared archetypes become shared seed profiles.
**Trade-offs:** Template YAML format changes. Existing templates.yaml needs migration.
**Sources:** templates.yaml (cartoon-villain, cartoon-hero, protector archetypes)
**Exploration:** quick
**Status:** captured

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

## D7: Implementation approach — rewrite SystemPromptRenderer

**Choice:** Approach A — rewrite SystemPromptRenderer.render() to produce minimal directive. Existing CognitionCore.promptSections() pipeline handles observation sections. New NeurocortexSeeder reads seed data from YAML and pushes into subsystems on first boot.
**Alternatives:**
- New CognitivePromptComposer alongside existing renderer — unnecessary indirection for single consumer.
- Invert pipeline (observation sections own everything including voice/style) — too risky without empirical evidence.
**Rationale:** Most direct path. CognitionCore already does the observation-side heavy lifting. We're removing competing directive content and ensuring subsystems are seeded so observations are never empty.
**Trade-offs:** SystemPromptRenderer changes affect all BriefingModes. Acceptable — pre-release, single consumer.
**Sources:** SystemPromptRenderer, CognitionCore.promptSections(), ConsolidationScheduler
**Exploration:** quick
**Depends on:** D1 (full rewrite scope), D2 (minimal directive), D5 (seeding mechanism)
**Status:** captured

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
