# HANDOFF — casehub-examples (slot 196)

## Last Session

Advanced queue from #823 to #520. Designed and planned the cognitive state dump endpoint (casehubio/neocortex#520). Blog SVG paths fixed.

### What was done

**1. Queue advanced: #823 → #520**

Closed #823 on GitHub (per-modality regulation eval — 4/4 tests passing). Advanced queue to casehubio/neocortex#520 (cognitive state dump endpoint).

**2. Blog image fix**

Fixed 3 broken SVG paths in `docs/blog/2026-11-09-mdp01-the-test-that-broke-the-math.md` — referenced `svg/` prefix but files are in blog root.

**3. Full design for #520**

Brainstormed, captured 5 decisions (D15-D19), ran light decision review (16 findings, 3 relevant to #520), revised 3 decisions:

- **D15 (revised):** PipelineTrace sub-record (not flat field explosion) — groups appraisal, CAPS, regulation, prompt sections
- **D16 (revised):** CognitionStateProvider SPI in cognitive-observability-api (not direct CognitionCore injection — fixes layer violation R1-04)
- **D17:** Live inspection + per-tick snapshot storage
- **D18:** New PipelineSnapshotStore with SQLite impl (not extend SnapshotStore)
- **D19 (revised):** CognitionCore captures CAPS data at call site, CapsEngine stays pure (R1-09)

Spec: `specs/issue-851-dynamic-regulatory-capacity/2026-11-09-cognitive-state-dump-design.md`

**4. Implementation plan written**

6 tasks across 3 batches. Plan: `plans/2026-11-09-cognitive-state-dump.md`

### Key discovery: CAPS settle() is consolidation-time

`CapsEngine.settle()` is called ONLY from `BehavioralSynthesisPhase` (consolidation), NOT from `CognitionCore.tick()`. CAPS data (input activations, settling result, attractors) updates on consolidation, not per-tick. The plan accounts for this:
- BehavioralSynthesisPhase caches last SettlingResult per agent
- CognitionCore queries it via the SPI implementation

### Module dependency finding

`cognitive-observability-api` and `cognition` don't depend on each other. For the SPI to work:
1. Add `cognition-api` and `caps-api` as dependencies of `cognitive-observability-api` (PipelineTrace needs AppraisalResult, SettlingResult, etc.)
2. Add `cognitive-observability-api` as a dependency of `cognition` (CognitionCore implements CognitionStateProvider)

These are all lightweight API modules — no implementation pulled in. Standard SPI pattern.

## Immediate Next Step

**Execute the implementation plan.** Start with `work continue`, read the plan at `plans/2026-11-09-cognitive-state-dump.md`, invoke `executing-plans`.

Tasks injected into `.queue`:
- [ ] Foundation: PipelineTrace + SPI + extend CognitionSnapshot
- [ ] Foundation: PipelineSnapshotStore + InMemory + contract test
- [ ] Storage: SqlitePipelineSnapshotStore
- [ ] Capture: CAPS caching + StateProvider impl + tick capture
- [ ] API: pipelineDump endpoint + CDI wiring
- [ ] Integration: CognitionStateProvider integration test

## What to Watch

1. **Module dependency wiring.** The first task must add pom.xml dependencies before writing code. Verify cognitive-observability-api can resolve cognition-api types (AppraisalResult, etc.) and that cognition can resolve cognitive-observability-api types (PipelineTrace, CognitionStateProvider).

2. **CognitionSnapshot construction sites.** Adding a `@Nullable PipelineTrace trace` field to the record breaks all existing construction sites. Use `ide_find_references` on the CognitionSnapshot constructor to find and fix every call site (add `, null` as last arg).

3. **BehavioralSynthesisPhase access from CognitionCore.** CognitionCore needs to query BehavioralSynthesisPhase for CAPS data. Check if CognitionCore already has a reference or needs one added via constructor/builder.

4. **Regulation result accessor.** The plan notes `EmotionRegulationTickParticipant` has a current result method — verify the exact method name with `ide_find_symbol` before writing the CognitionStateProvider implementation.

5. **Prompt section rendering.** CognitionCore.promptSections() returns renderers, not rendered text. The CognitionStateProvider implementation needs to call render() on each with a CognitionRenderContext. Check how existing code constructs this context.

## Queue Status

- [x] #823 — Per-modality (closed)
- [ ] #520 — Cognitive (active, implementation plan ready)
- [ ] #203 — Spatial
- [ ] #806 — CAPS
- [ ] #315 — Cognitive (casehub-mosaic)
- [ ] #828 — Regulation predictor observability
- [ ] #818 — Automatic

## References

- Spec: `specs/issue-851-dynamic-regulatory-capacity/2026-11-09-cognitive-state-dump-design.md`
- Plan: `plans/2026-11-09-cognitive-state-dump.md`
- Decisions: `specs/issue-851-dynamic-regulatory-capacity/decisions.md` (D15-D19)
- Decision review: `/Users/mdproctor/reviews/casehub-slots/issue-851-decision-20261109-021336/`
- Pipeline state: `specs/issue-851-dynamic-regulatory-capacity/pipeline.state` (PLANNING)
- Blog fix: `examples/docs/blog/2026-11-09-mdp01-the-test-that-broke-the-math.md` (SVG paths corrected)
