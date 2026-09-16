# Directive-Minimal Architecture Design

**Issue:** casehubio/blocks#283
**Branch:** issue-63-fix-cdi-config-beans
**Date:** 2026-09-16

## Problem

The current system prompt places cognitive data (goals, beliefs, strategies, constraints, disposition) alongside identity in the agent's briefing directive. The LLM treats system prompt content as authoritative instruction, overpowering dynamic cognitive sections from neurocortex subsystems that appear in the user prompt. Mood shifts, drive intensities, learned strategies, and trust perceptions read as supplementary context rather than active cognitive state.

The duplication compounds the problem: goals appear in both the system prompt (from eidos descriptor) and observation sections (from `CognitiveObservationSections.goalsSection()` and `GoalPromptSection`). Constraints appear in both channels. The directive version dominates, making the observation-side rendering decorative rather than functional.

## Architecture

### Before (current)

```
System prompt (directive):
  ├── Briefing text (identity + behavioral instructions + character knowledge)
  ├── Templates (voice/style + behavioral patterns)
  ├── Goals (static, from descriptor YAML)
  ├── Constraints (all severities)
  └── Disposition (MBTI, Enneagram)

User prompt (observations):
  ├── World state (room, characters, objects)
  ├── Cognitive sections (CognitionCore.promptSections())
  │     ├── Mood (MoodOrchestrator)
  │     ├── Drives (DriveOrchestrator)
  │     ├── Narrative (NarrativeOrchestrator)
  │     ├── Mental model / beliefs (MentalModelOrchestrator)
  │     ├── User model (UserModelOrchestrator)
  │     ├── Strategy (StrategyLearningOrchestrator)
  │     ├── Goals (GoalProposalOrchestrator)  ← DUPLICATE
  │     ├── Personality (disposition)         ← DUPLICATE
  │     └── Constraints                       ← DUPLICATE
  ├── CharacterCognition sections
  │     ├── Drives (from social-config.yaml)  ← DUPLICATE of DriveOrchestrator
  │     ├── Beliefs (from social-config.yaml) ← DUPLICATE of MentalModelOrchestrator
  │     ├── Norms (from social-config.yaml)
  │     ├── Trust perceptions (from mindmap overlay)
  │     └── Social awareness (CognitiveProfile.compare())
  ├── Memories, reflections, insights
  └── Recent activity, plans, inventory
```

### After (proposed)

```
System prompt (minimal directive):
  ├── Identity: name, role, voice
  ├── Hard constraints (HARD severity only)
  ├── Templates (voice/style — unchanged)
  └── Cognitive preamble (auto-generated: "you have a brain, here's how to use it")

User prompt (observations — sole source of cognitive state):
  ├── World state (unchanged)
  ├── Cognitive sections (CognitionCore.promptSections())
  │     ├── Mood, Drives, Narrative, Mental model, User model, Strategy, Goals
  │     ├── Personality
  │     └── Soft constraints
  ├── CharacterCognition sections (deduplicated)
  │     ├── Trust perceptions
  │     ├── Social awareness
  │     └── Norms
  ├── Memories, reflections, insights
  └── Recent activity, plans, inventory
```

## Components

### 1. CognitiveSystemPromptRenderer

**Location:** `blocks-core` (or `blocks-agentic` — wherever the cognitive pipeline lives)
**Package:** `io.casehub.blocks.agentic.social.prompt`

A `@Alternative @Priority(1)` CDI bean implementing `SystemPromptRenderer`. When present on the classpath, it overrides `EidosSystemPromptRenderer`. The eidos renderer stays untouched — no eidos changes.

**Renders:**
1. **Identity block:** Agent name + role sentence. Extracted from `AgentDescriptor.briefing()` — the spec defines a new `identity` field in the descriptor YAML (see §4 below), or the renderer extracts the first sentence(s) of the briefing that establish identity.
2. **Voice block:** Speaking style, mannerisms, catchphrases. From descriptor `briefing` field (stripped of behavioral instructions and character knowledge) plus template content (templates are pure voice/style).
3. **Hard constraints:** Only constraints with `severity: HARD` from `AgentDescriptor.constraints()`.
4. **Cognitive preamble:** Auto-generated paragraph from active subsystems (see §2 below).

**Does NOT render:** goals, soft constraints, disposition, drives, beliefs, strategies, narrative, or any dynamic cognitive data.

**RenderedPrompt compatibility:** Returns a `RenderedPrompt` with `format: MARKDOWN`, `enriched: false` (no semantic enrichment — that's an eidos feature), and a valid `descriptorHash`/`contextHash` for caching.

### 2. Cognitive Preamble Generator

**Location:** `blocks-core`, same package as `CognitiveSystemPromptRenderer`
**Class:** `CognitivePreambleGenerator`

Inspects which `CognitionCore` subsystems are active (via config flags: `config.moodEnabled()`, `config.drivesEnabled()`, etc.) and composes a single coherent paragraph that tells the agent how to use its cognitive faculties.

Example output when mood, drives, mental model, goals, and strategy are active:

> You have an inner life. Your emotional state colours how you respond — let it shape your tone and choices. You have motivational drives that pull at you; some are stronger than others right now. You hold beliefs about the people around you, formed from observation — act on them, update them when evidence contradicts. You have goals that emerged from your motivations — pursue them, reprioritise when circumstances change. You have learned strategies from past interactions — apply what worked, abandon what didn't.

This replaces `DirectiveSection` per-section wrapping. When `CognitiveSystemPromptRenderer` is active, `config.directivePrompts()` should be `false` — the preamble provides the meta-instruction once in the system prompt rather than prepending it to each observation section.

### 3. Neurocortex Seeding

**Generalises:** `ManorCognitiveSeeder` (currently in wacky-manor, manually instantiated)

The seeder reads seed data from character YAML and pushes initial state into cognitive subsystems on first boot (when subsystem state is empty).

**Seed sources (per character):**
- `social-config.yaml` → drives, norms, initial-beliefs, relationships (already exists)
- `descriptors-composite.yaml` → goals (currently in descriptor, moving to seed-only)
- `trust-evolution.yaml` → trust config (already exists, already seeded)

**Seed targets:**
| Source | Target subsystem | Already seeded? |
|--------|-----------------|----------------|
| `initial-beliefs` | MindMapStore (belief nodes) | Yes — ManorCognitiveSeeder |
| `relationships` | MindMapStore (person-entity overlays with PAD) | Yes — ManorCognitiveSeeder.seedPeople() |
| `drives` | DriveOrchestrator / DriveProfile | No — currently read from social-config at render time |
| `norms` | Norms rendering | No — currently read from social-config at render time |
| `goals` | GoalProposalOrchestrator | No — currently from descriptor YAML |

**Key change for drives and norms:** Currently `CharacterCognition.renderCognitiveSections()` reads drives and norms from `SocialConfig` at render time and produces observation text directly. In the new model, drives and norms are seeded into their respective subsystems at boot, and `CognitionCore.promptSections()` renders them — the same path as mood, strategy, and mental model.

This requires:
- `DriveOrchestrator` (or a new `DriveProfile` store) accepts seeded drive state
- A norms section in `CognitionCore.promptSections()` (currently missing — norms only come from `CharacterCognition`)
- `CharacterCognition.renderCognitiveSections()` stops rendering drives, norms, beliefs, and constraints directly — these come from CognitionCore

**Seeder lifecycle:** Called once per agent at scenario bootstrap (same point where `ManorCognitiveSeeder.seed()` and `seedPeople()` are called today). Idempotent — if subsystem state already exists, seeding is skipped.

**Architecture question — seeder location:**
- The seeder SPI/interface belongs in `blocks-core` (alongside CognitionCore)
- The implementation that reads from `SocialConfig` + `AgentDescriptor` stays in wacky-manor (application-specific YAML formats)
- ManorCognitiveSeeder is enhanced rather than replaced — it already handles beliefs and relationships, it gains drives, norms, and goals

### 4. Descriptor YAML Changes

**`descriptors-composite.yaml`:**

Current briefing text mixes identity, voice, behavioral instructions, and character knowledge. Example (Penelope):

> You are Penelope Pitstop — a glamorous, sweet-natured Southern belle... You speak with a Southern drawl and use phrases like "Why, how delightful!"... You know Sylvester Sneekly as a helpful and charming estate manager.

In the new model, the `briefing` field is rewritten to contain only identity and voice. The eidos `AgentDescriptor` model is unchanged — `briefing` remains a single string field. The `CognitiveSystemPromptRenderer` consumes the same `AgentDescriptor`; the change is in what content authors put in the `briefing` field and how the renderer uses it.

```yaml
- agentId: penelope-pitstop
  name: Penelope Pitstop
  briefing: >-
    A glamorous, sweet-natured Southern belle and heiress.
    You speak with a Southern drawl and use phrases like
    "Why, how delightful!" and "Oh my stars!"
  # goals: REMOVED — seeded into GoalProposalOrchestrator via social-config.yaml
  constraints:
    - {name: never-break-cover, description: "...", severity: HARD}  # stays in directive
    # SOFT constraints move to social-config.yaml as norms or seeded constraints
  templates: [{ref: hanna-barbera-cartoon-style}]  # voice/style — stays
  disposition:  # kept in YAML but NOT rendered in directive — seeded into subsystems
    mbtiType: ESFJ
    enneagramType: helper
```

**Character knowledge** ("You know Sylvester Sneekly as a helpful estate manager") becomes a seeded belief in `social-config.yaml`:

```yaml
penelope-pitstop:
  initial-beliefs:
    - {key: sneekly-identity, value: "Sylvester Sneekly is a helpful and charming estate manager"}
```

### 5. Deduplication

Remove all observation-side content that duplicates what CognitionCore already renders:

| Currently duplicated | Remove from | Keep in |
|---------------------|-------------|---------|
| Goals | System prompt (descriptor) + `CognitiveObservationSections.goalsSection()` | `GoalPromptSection` (CognitionCore) |
| Constraints | System prompt + `CharacterCognition` | System prompt (HARD only) + `ConstraintPromptSection` (SOFT, via CognitionCore) |
| Drives | `CharacterCognition` direct render | `DrivePromptSection` (CognitionCore) |
| Beliefs | `CharacterCognition` direct render from social-config | `MentalModelPromptSection` (CognitionCore) |
| Personality/disposition | System prompt | `PersonalityPromptSection` (CognitionCore) |

After deduplication, `CharacterCognition.renderCognitiveSections()` is significantly slimmed — it retains only:
- Trust perceptions (from mindmap overlay — no CognitionCore equivalent)
- Social awareness (perspectival comparison — no CognitionCore equivalent)
- Norms (until a `NormsPromptSection` is added to CognitionCore)

### 6. DirectiveSection Deprecation

`DirectiveSection` wrapping (`config.directivePrompts()`) is replaced by the cognitive preamble in the system prompt. When `CognitiveSystemPromptRenderer` is active:
- `config.directivePrompts()` returns `false`
- `CognitionCore.promptSections()` returns unwrapped sections
- Each section's heading and structure must be self-explanatory (section headings like "Your Current Mood", "Your Drives", "What You Believe" already achieve this)

The `DirectiveSection` class is not deleted — it remains available for non-cognitive rendering paths.

## Cross-Repo Scope

| Repo | Changes | Issue |
|------|---------|-------|
| **blocks** | `CognitiveSystemPromptRenderer`, `CognitivePreambleGenerator`, seeder SPI, `CognitionCore` config changes, norms prompt section | casehubio/blocks#283 |
| **examples/wacky-manor** | Descriptor YAML restructure, `ManorCognitiveSeeder` enhancement, `CharacterCognition` deduplication, `ScenarioOrchestrator` wiring | New issue on casehubio/examples |
| **neocortex** | No changes expected — consolidation pipeline already observation-side | None |

## Empirical Validation

This architectural change alters how the LLM processes agent personality. Validation requirements:

1. **Constraint adherence:** Hard constraints in system prompt must be followed as reliably as before. Test with Hooded Claw's `never-break-cover` constraint.
2. **Cognitive section influence:** Mood, drives, and trust perceptions should visibly influence agent behavior (not just appear in the prompt). Compare agent responses before/after the change.
3. **Cold-start quality:** Seeded initial state must produce agent behavior comparable to current briefing-heavy prompts.
4. **Multi-model sensitivity:** If eidos supports multiple LLM backends, verify that minimal-directive + rich-observations works across model families. System prompt vs user-turn attention weight varies by model.

The LLM eval framework (existing `llm-eval` Maven profile) can be extended with tests for these dimensions.

## References

- `EidosSystemPromptRenderer` — `casehub-eidos/runtime/renderer/EidosSystemPromptRenderer.java`
- `SystemPromptRenderer` SPI — `casehub-eidos-api`
- `CognitionCore.promptSections()` — `blocks-core/.../social/CognitionCore.java:344`
- `DirectiveSection` — `blocks-core/.../social/prompt/DirectiveSection.java`
- `ManorCognitiveSeeder` — `wacky-manor/.../agent/ManorCognitiveSeeder.java`
- `CharacterCognition` — `wacky-manor/.../agent/CharacterCognition.java`
- `ScenarioOrchestrator.renderPrompt()` — `wacky-manor/.../agent/ScenarioOrchestrator.java:543`
- `descriptors-composite.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/`
- `social-config.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/`
- `templates.yaml` — `wacky-manor/src/main/resources/META-INF/eidos/`
- Decision review findings — `reviews/casehub-slots/blocks-283-decision-20260916-104401/`
- Decisions — `specs/issue-63-fix-cdi-config-beans/decisions.md`
