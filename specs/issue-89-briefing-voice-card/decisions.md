# Decisions — #89 Redefine Briefing as Voice Card

## D1: Briefing field structure

**Choice:** Structured sub-fields in the eidos descriptor — split the briefing into explicit voice profile fields with distinct YAML keys.
**Alternatives:**
- Single text field (rewrite content only) — simpler but the constraint is social (authoring convention) not structural; doesn't prevent behavioral instructions creeping back
- Hybrid (voice field + briefing) — half-measure, two fields but no full separation
**Rationale:** Pre-release platform — the right time to harden foundations. Structural separation enforces the voice/behavior boundary at the API level, not just by convention. Follows the same principle as the write-content skill's Form/Mode/Voice taxonomy: what the agent is, how it behaves, and how it sounds are different concerns that should be structurally distinct.
**Trade-offs:** Requires eidos API changes (new record fields, renderer updates, vocabulary support). Every consuming app's descriptor YAML needs updating.
**Sources:** write-content skill (Form/Mode/Voice taxonomy), mark-proctor-voice.md (voice fingerprint structure), descriptors-composite.yaml (current briefing content)
**Exploration:** quick
**Status:** captured

## D2: Voice profile fields — pure structured, no freeform escape hatch

**Choice:** Pure structured fields with no prose escape hatch. Fields: `register`, `accent`, `catchphrases`, `speech_patterns`, `vocabulary_uses`, `vocabulary_avoids`, `quirks`, `personas` (optional map of named voice variants).
**Alternatives:**
- Prose block (single text field named 'voice') — simpler authoring but no programmatic hooks, and authors will dump behavioral instructions into it
- Hybrid (structured + freeform voice_notes) — escape hatch would be used to circumvent the separation, defeating the design
**Rationale:** Every piece of voice content from the four analyzed briefings decomposed cleanly into structured dimensions. Nothing was genuinely irreducible to structure. Content that *feels* like it needs prose is either a structured voice dimension or a behavioral instruction that belongs in the cognitive layer. The structural constraint IS the design — it forces the separation.
**Trade-offs:** Authors cannot write freeform voice descriptions. If a genuinely novel voice dimension is discovered, a new field must be added to the API rather than worked around in prose.
**Sources:** Analysis of Hooded Claw (dual-persona decomposes into personas map + cognitive switching), Penelope (Southern drawl = accent + catchphrases), Ant Hill Mob (ensemble = quirks), Dick Dastardly (dramatic register + catchphrases)
**Exploration:** deep-analysis
**Status:** captured

## D3: Hooded Claw dual-voice — personas map with cognitive switching

**Choice:** Multi-voice characters use a `personas` map in the voice profile (e.g., `sneekly: {register: obsequious, ...}`, `claw: {register: grandiose, ...}`). The switching logic ("Sneekly when others present, Claw when alone") lives in the cognitive layer via existing HARD constraints and goals.
**Alternatives:**
- Single voice with prose instructions for switching — conflates voice and behavior
- Separate descriptors per persona — over-engineers identity; the Hooded Claw is one agent with two modes, not two agents
**Rationale:** The switching rule is cognitive behavior (driven by `never-break-cover` constraint and `maintain-disguise` goal). The two voices are independently structurable. Decomposing this way uses existing cognitive infrastructure rather than adding voice-layer complexity.
**Trade-offs:** Requires the cognitive system to signal which persona is active so the renderer can select the right voice variant. This signal path doesn't exist yet.
**Sources:** Hooded Claw briefing analysis, CognitionCore constraint handling, AgentConstraint.HARD severity
**Depends on:** D2 (voice profile fields)
**Exploration:** deep-analysis
**Status:** captured

## D4: Architecture — clean two-layer split, not a unified pipeline

**Choice:** No single composition pipeline. Clean the existing two-layer architecture (system prompt + observation) so each layer has clear, non-overlapping responsibilities. Eidos owns the system prompt (identity). Blocks/app owns the observation (cognitive state).
**Alternatives:**
- Blocks owns unified pipeline — would need eidos runtime dependency, doing eidos's job
- Eidos owns unified pipeline — would need to know about cognitive systems, inverts dependency direction
- New composition module — adds complexity for something the app already does
**Rationale:** The two-layer split maps to the LLM API contract: system prompt is cached by the provider, observation varies per tick. The problem isn't the absence of a unified pipeline — it's that the layers have unclear boundaries and duplicate content. Fix the boundaries, don't add a new abstraction.
**Trade-offs:** No single place that "owns" the full prompt composition — the app remains the point where the two layers meet. But this is inherent to the architecture and trying to centralize it creates worse problems.
**Sources:** LLM API caching contract, eidos/blocks dependency graph analysis, current duplication in PersonalityPromptSection/ConstraintPromptSection/goals
**Exploration:** deep-analysis
**Status:** captured

## D5: Remove duplication between layers

**Choice:** Kill `PersonalityPromptSection` (eidos already renders vocabulary-resolved personality in system prompt). Kill `ConstraintPromptSection` (constraints are static identity, belong in system prompt). Distinguish authored goals (eidos descriptor) from emergent goals (GoalPromptSection) with clear naming.
**Alternatives:**
- Keep duplicates with deduplication logic — complexity without benefit, symptoms not cause
- Move all to observation — loses system prompt caching benefit for static content
**Rationale:** Each piece of content should have exactly one owner. Personality is identity (eidos). Constraints are identity (eidos). Emergent goals are cognitive state (blocks). Authored goals are identity (eidos). The raw-codes PersonalityPromptSection was always a workaround for blocks not having vocabulary resolution — the right fix is to let eidos handle personality in the system prompt.
**Trade-offs:** Removing PersonalityPromptSection means blocks-core loses the ability to render personality independently. Apps that don't use eidos won't get personality in their prompt. Acceptable because personality IS an eidos concept.
**Sources:** CognitionCore.promptSections() line 377-413, PersonalityPromptSection (raw DispositionValue codes), EidosRenderPipeline (vocabulary resolution)
**Depends on:** D4 (two-layer split)
**Exploration:** deep-analysis
**Status:** captured

## D6: Cognitive preamble bridge

**Choice:** CognitionCore exposes preamble text (from CognitivePreambleGenerator). The app passes it to eidos as extension data, and the eidos renderer includes it in the system prompt.
**Alternatives:**
- Preamble in observation — wrong layer; it's architecture framing, not per-tick state
- Eidos generates preamble itself — eidos doesn't know what cognitive systems are enabled
**Rationale:** The preamble describes what cognitive subsystems are active ("You have an inner life. Your emotional state colours how you respond..."). This is stable across a session (config-dependent, not tick-dependent) and belongs in the system prompt. But only blocks knows what's enabled. The bridge via extension data respects the dependency direction: blocks → eidos-api (which already exists).
**Trade-offs:** Adds a coupling point where the app must wire preamble from CognitionCore into eidos's render call. But this is explicit and traceable, not hidden.
**Sources:** CognitivePreambleGenerator.java (currently unused in rendering path), AgentDescriptor.extensionData() (existing extension mechanism)
**Depends on:** D4 (two-layer split)
**Exploration:** quick
**Status:** captured

## D7: Implementation scope — validate with 3-4 characters

**Choice:** Validate the design with 3-4 contrasting characters (Hooded Claw for dual-persona, Penelope for straightforward voice, Ant Hill Mob for ensemble, one more). Remaining 13 characters as a mechanical follow-up issue.
**Alternatives:**
- All 17 characters in this issue — more work, delays validation feedback
- Design only, no rewrites — misses the concrete validation that proves the design works
**Rationale:** The design needs empirical validation before scaling. Contrasting characters test the edge cases (dual-persona, ensemble, straightforward). Once validated, the remaining rewrites are mechanical.
**Trade-offs:** Two issues instead of one. The follow-up issue is low-risk but still work.
**Sources:** descriptors-composite.yaml (17 characters with varying complexity)
**Exploration:** quick
**Status:** captured

## D8: Emergence verification — baseline then compare

**Choice:** Extend the Phase D eval infrastructure. Run 1: full briefings with full cognitive config → persist output to git. Run 2: voice-only briefings with same config → compare delta. This gives concrete evidence of what behavior was briefing-driven vs cognition-driven.
**Alternatives:**
- Separate emergence eval suite — duplicates infrastructure
- Manual verification only — no quantitative comparison, no reproducibility
**Rationale:** The Phase D eval infrastructure (CognitiveEvalTest, progressive config levels, delta capture) already exists. No prior eval output was preserved (target/ is ephemeral). Establishing a baseline first, then comparing voice-only, directly answers the question "does the cognitive system produce distinct behavior without behavioral scripting?"
**Trade-offs:** Two eval runs required (full-briefing baseline + voice-only). LLM eval output is non-deterministic, so the comparison is qualitative delta analysis, not exact diff.
**Sources:** CognitiveEvalTest.java (existing infrastructure), Phase D spec §5 (eval suite design)
**Depends on:** D7 (implementation scope — uses the same 3-4 characters)
**Exploration:** quick
**Status:** captured
