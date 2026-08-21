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
