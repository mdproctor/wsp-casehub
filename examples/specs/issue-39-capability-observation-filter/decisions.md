# Decisions — #39 Capability-Driven Observation Filtering

## D1: Filtering operations scope

**Choice:** Three composable operations — resolution control (same event, different detail), visibility gating (sections hidden for agents lacking capabilities), and interpretive framing (same data, different analytical lens). All three live in blocks as pipeline operations.
**Alternatives:**
- Resolution only — matches the manor's current pattern but too narrow for enterprise/simulation use cases
- Resolution + visibility only — misses the interpretive framing that simulation and adversarial scenarios need
**Rationale:** A universal observation system must handle all three. Enterprise agents need visibility gating (compliance vs scheduling). Simulations need resolution (information asymmetry). Adversarial and analytical agents need interpretive framing. The manor becomes a thin application layer.
**Trade-offs:** More complex pipeline design. Acceptable — the operations compose cleanly and each can be used independently.
**Exploration:** quick
**Status:** captured

## D2: Filtering model — metadata on sections

**Choice:** Sections carry metadata (required tags, resolution tier). The provider emits all sections at maximum resolution, observer-agnostic. A pipeline filter strips or downgrades based on the observer's capabilities. The provider never branches on observer tags.
**Alternatives:**
- Pre-section filtering (provider receives observer capabilities and makes filtering decisions internally) — what the manor does today. Forces every provider to implement filtering logic, couples provider to observer knowledge, prevents composable filtering.
**Rationale:** Observer-agnostic providers describe the world at full fidelity. Filtering is a separate, composable concern in the pipeline. ManorWorldObservationProvider would emit keen observations unconditionally with `requires: ["perception"]` — the pipeline strips them for non-perceptive observers. Clean separation of "what exists" from "who can see it."
**Trade-offs:** ObservationSection gains annotation surface. Providers must learn to annotate sections. Existing providers work unchanged (unannotated sections pass through all filters).
**Depends on:** D1 (three operations — the metadata model must support all three)
**Exploration:** quick
**Status:** captured

## D3: Metadata carrier — wrapper record

**Choice:** `AnnotatedSection` wrapper record that pairs any `ObservationSection` with filtering metadata (required tags, resolution tier, interpretive frame). Providers that don't filter return bare sections. Providers that do return `AnnotatedSection`. The pipeline unwraps after filtering. The `ObservationSection` sealed interface stays untouched.
**Alternatives:**
- Extend the sealed interface with metadata fields — changes the pattern-match surface that `AffordanceRenderer` uses, forces every existing section construction site to update, couples rendering to filtering concerns
**Rationale:** Additive. Existing code constructing bare `ObservationSection` works unchanged. The pipeline handles both bare sections (pass-through) and annotated sections (apply filters). The renderer only sees unwrapped sections — metadata is a pipeline concern, not a rendering concern.
**Trade-offs:** Two types in the pipeline (`ObservationSection` and `AnnotatedSection`) — callers must choose which to emit. Acceptable — the default (bare section) is the simple path.
**Depends on:** D2 (metadata on sections model)
**Exploration:** quick
**Status:** captured
