# Decisions — #48 Extract Observation SPI

## D1: Cognitive section formatter location

**Choice:** blocks — in the `io.casehub.blocks.summarisation.observation.affordance` package alongside `ObservationSection` and `WorldObservationProvider`. A utility class (e.g. `CognitiveObservationSections`) provides standard factory methods for producing `ObservationSection` instances from platform cognitive types (`AgentGoal`, `Memory`, `PartitionedDrain`).
**Alternatives:**
- quarkmind-core — conceptual fit (agent cognition), but quarkmind-core is intentionally lean (engine-api + eidos-api only), doesn't know about blocks, and adding blocks + neocortex-memory-api dependencies would widen the core for 5 utility methods
- Stay in the manor — defeats the goal of making cognitive sections reusable across agents
**Rationale:** The formatters are `T → ObservationSection` factory methods — they belong with the type they produce. blocks already depends on eidos-api and neocortex-memory-api, so no new dependencies needed. Can be moved to quarkmind-core later if quarkmind grows toward prompt assembly responsibility.
**Trade-offs:** blocks gains cognitive awareness (goals, memories, reflections). Acceptable — it already depends on those APIs.
**Exploration:** deep-analysis
**Status:** captured

## D2: Method categorisation

**Choice:** Three-way split by type dependency:
- **World sections (7)** → `ManorWorldObservationProvider` in the manor: `locationSection`, `exitsSection`, `objectsSection` + `toObservableEntity`, `charactersSection`, `keenObservationsSection`, `directedDialogueSection`, `rememberedSection`
- **Cognitive sections (5)** → `CognitiveObservationSections` in blocks: `goalsSection`, `recentActivitySection`, `pastExperienceSection`, `insightsSection`, `relationshipNotesSection`
- **CharacterState-dependent (4)** → stay in manor's `ObservationBuilder`: `inventorySection`, `currentThinkingSection`, `planSections`, `lastActionResultSection`
**Alternatives:**
- Two-way split (world vs everything else stays in manor) — misses the cognitive extraction goal
- Two-way split (world vs all cognitive to blocks) — CharacterState is a manor type; blocks can't depend on it
**Rationale:** The split follows the type boundary: methods using only platform types (AgentGoal, Memory, PartitionedDrain) move to blocks; methods requiring manor types (CharacterState, WorldState) stay in the manor. inventorySection reads CharacterState.inventory() — it's character-own-state, not world perception, so it stays in manor.
**Trade-offs:** 4 methods remain in the manor's ObservationBuilder. Extracting them would require abstracting CharacterState — deferred.
**Depends on:** D1 (cognitive formatters go to blocks)
**Exploration:** quick
**Status:** captured
