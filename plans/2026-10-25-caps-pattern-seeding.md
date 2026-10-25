# Import caps_patterns as AgentCapsState Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** neocortex#685 — Import caps_patterns as AgentCapsState — seed CAPS weights from biography

**Goal:** Replace inert MindMap property storage of caps_patterns with actual CAPS engine weight seeding, so biographical patterns shape behavioral settling from the first evaluation.

**Architecture:** Enhance the `CapsPattern` record to include structured `inputActivations` (a map of topology input node IDs to activation levels) and `confidence`. Update the Python enrichment prompt to produce these structured activations. In `EntityEnrichmentHandler`, inject `CapsEngine`, read the agent's disposition from the `BiographyProfile`, call `initializeAgent()` once, then `updateWeights()` per pattern to build a biographical baseline.

**Tech Stack:** Java 26 (records, sealed), Quarkus CDI, Python 3 (extraction pipeline), Jackson YAML/JSON, JUnit 5 + AssertJ

## Global Constraints

- IntelliJ MCP for all code navigation and structural editing
- TDD: failing test before implementation
- All caps_patterns field changes must remain backward-compatible (new fields optional)
- Topology input node IDs are the canonical vocabulary — no free-form trigger terms in `input_activations`
- `memory-seeding` gains a compile dependency on `caps-api` and test dependency on `caps-testing`

---

## Batch 1: Schema extension + Python pipeline

### Task 1: Extend CapsPattern and convert_corpus.py

**Files:**
- Modify: `memory-seeding/src/main/java/io/casehub/neocortex/memory/seeding/biography/EntityEnrichmentEntry.java:22` (CapsPattern record)
- Modify: `extraction/scripts/convert_corpus.py:350-354` (caps_patterns passthrough)
- Modify: `extraction/tests/test_convert_corpus.py:456-467` (Python test)

**Interfaces:**
- Produces: `CapsPattern(String pattern, Double frequency, Double confidence, Map<String, Double> inputActivations)` — used by Task 3 in EntityEnrichmentHandler

- [ ] **Step 1: Write the failing Python test**

Add a test to `test_convert_corpus.py` that verifies `input_activations` and `confidence` are passed through:

```python
def test_caps_patterns_with_input_activations(self):
    enrichments = {
        "Frida Kahlo": {
            "caps_patterns": [
                {
                    "pattern": "betrayal → trust_violation → AUTONOMY_ASSERTION",
                    "frequency": 3,
                    "confidence": 0.87,
                    "input_activations": {"betrayal": 0.9, "abandonment": 0.6},
                },
            ],
        },
    }
    _, files = _convert(_corpus(entity_enrichments=enrichments))
    entry = files["entity-enrichments.yaml"]["entries"][0]
    assert entry["caps_patterns"][0]["confidence"] == 0.87
    assert entry["caps_patterns"][0]["input_activations"] == {"betrayal": 0.9, "abandonment": 0.6}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest extraction/tests/test_convert_corpus.py::TestEntityEnrichments::test_caps_patterns_with_input_activations -v`
Expected: FAIL — `input_activations` and `confidence` are stripped by convert_corpus.py

- [ ] **Step 3: Update convert_corpus.py to pass through new fields**

In `extraction/scripts/convert_corpus.py`, change the caps_patterns dict comprehension (lines 350-354) from:

```python
entry["caps_patterns"] = [
    {"pattern": p.get("pattern", ""), "frequency": p.get("frequency")}
    for p in data["caps_patterns"]
]
```

to:

```python
entry["caps_patterns"] = [
    {
        "pattern": p.get("pattern", ""),
        "frequency": p.get("frequency"),
        "confidence": p.get("confidence"),
        "input_activations": p.get("input_activations"),
    }
    for p in data["caps_patterns"]
]
```

- [ ] **Step 4: Run Python tests to verify they pass**

Run: `python3 -m pytest extraction/tests/test_convert_corpus.py::TestEntityEnrichments -v`
Expected: PASS (both old and new tests)

- [ ] **Step 5: Update CapsPattern Java record**

In `EntityEnrichmentEntry.java`, change the `CapsPattern` inner record from:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record CapsPattern(String pattern, Double frequency) {}
```

to:

```java
@JsonIgnoreProperties(ignoreUnknown = true)
public record CapsPattern(
    String pattern,
    Double frequency,
    Double confidence,
    @JsonProperty("input_activations") Map<String, Double> inputActivations
) {}
```

Add `import java.util.Map;` if not already present (it is — used by `driveBaselines`).

- [ ] **Step 6: Update existing Java test for new constructor**

In `EntityEnrichmentHandlerTest.java`, update the `storesCapsPatterns` test (line 67) — the constructor call must match the new 4-param record:

```java
var patterns = List.of(
        new EntityEnrichmentEntry.CapsPattern("threat -> creative", 0.8, null, null));
```

Also update `EntityEnrichmentEntry` constructor calls that pass `null` for capsPatterns — these won't break since they pass `null` for the whole list, not individual CapsPattern instances.

- [ ] **Step 7: Build and run Java tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f memory-seeding/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS — all existing tests compile and pass with the new record shape

- [ ] **Step 8: Commit**

```bash
git add memory-seeding/src/main/java/io/casehub/neocortex/memory/seeding/biography/EntityEnrichmentEntry.java
git add extraction/scripts/convert_corpus.py extraction/tests/test_convert_corpus.py
git add memory-seeding/src/test/java/io/casehub/neocortex/memory/seeding/biography/EntityEnrichmentHandlerTest.java
git commit -m "feat(#685): extend CapsPattern with inputActivations and confidence

Add structured input_activations (Map<String, Double>) and confidence
fields to CapsPattern record. Update convert_corpus.py to pass these
through from enrichment data. Backward-compatible — both fields are
optional.

Refs casehubio/neocortex#685

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Update enrichment prompt with topology node IDs

**Files:**
- Modify: `extraction/scripts/enrich_entity.py:25-38` (DRIVES_CAPS_SYSTEM prompt)

**Interfaces:**
- Produces: enrichment output with `input_activations: {nodeId: activation}` per caps_pattern — consumed by convert_corpus.py (Task 1) and eventually EntityEnrichmentHandler (Task 3)

- [ ] **Step 1: Identify topology input node IDs**

The canonical set from `caps-topology.yaml` (30 nodes across 7 categories):

```
relationship: secure_attachment, inconsistent_care, rejection, betrayal, abandonment, neglect
threat: physical_threat, social_threat, psychological_threat, unpredictable_danger
achievement: mastery, failure, performance_judged, competence_recognition
social: acceptance, exclusion, dominance, submission, modeling_observed
resource: scarcity, abundance, competition, sharing
autonomy: agency_granted, controlled, choice_available, powerlessness
consequence: reward, punishment, reward_removal, aversive_removal
```

- [ ] **Step 2: Update DRIVES_CAPS_SYSTEM prompt**

In `extraction/scripts/enrich_entity.py`, replace the `DRIVES_CAPS_SYSTEM` constant (lines 25-38) with:

```python
DRIVES_CAPS_SYSTEM = """You are a cognitive psychologist assessing an entity's motivational drives and situational activation patterns from biographical data.

### 1. Drive Profile (4 axes)
For each axis, provide a BASELINE level (not a delta) and justification:
- curiosity: baseline 0.0-1.0 + justification
- competence: baseline 0.0-1.0 + justification
- affiliation: baseline 0.0-1.0 + justification
- autonomy: baseline 0.0-1.0 + justification

### 2. CAPS Patterns
Identify recurring activation patterns: what situations consistently trigger what responses?

Each pattern MUST include:
- pattern: narrative description "trigger → mediating → RESPONSE"
- frequency: how often this pattern fires (1-5)
- confidence: how certain the pattern is (0.0-1.0)
- input_activations: a map of CAPS input node IDs to activation levels (0.0-1.0)

Valid input node IDs (use ONLY these):
  relationship: secure_attachment, inconsistent_care, rejection, betrayal, abandonment, neglect
  threat: physical_threat, social_threat, psychological_threat, unpredictable_danger
  achievement: mastery, failure, performance_judged, competence_recognition
  social: acceptance, exclusion, dominance, submission, modeling_observed
  resource: scarcity, abundance, competition, sharing
  autonomy: agency_granted, controlled, choice_available, powerlessness
  consequence: reward, punishment, reward_removal, aversive_removal

Format: [{"pattern": "betrayal → trust_violation → AUTONOMY_ASSERTION", "frequency": 3, "confidence": 0.87, "input_activations": {"betrayal": 0.9, "abandonment": 0.6}}]

Respond with JSON only: {"drive_baselines": {...}, "caps_patterns": [...]}"""
```

- [ ] **Step 3: Verify prompt loads correctly**

Run: `python3 -c "import sys; sys.path.insert(0, 'extraction/scripts'); from enrich_entity import DRIVES_CAPS_SYSTEM; print('OK:', len(DRIVES_CAPS_SYSTEM), 'chars')"`
Expected: OK with character count

- [ ] **Step 4: Commit**

```bash
git add extraction/scripts/enrich_entity.py
git commit -m "feat(#685): update CAPS enrichment prompt for structured input_activations

Provide the 30 topology input node IDs in the enrichment prompt so the
LLM produces structured input_activations (map of nodeId → activation)
alongside the narrative pattern string. This closes the loop between
enrichment output and CAPS engine input.

Refs casehubio/neocortex#685

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: CAPS seeding in importer

### Task 3: EntityEnrichmentHandler CAPS seeding

**Files:**
- Modify: `memory-seeding/pom.xml` (add caps-api compile dep, caps-testing test dep)
- Modify: `memory-seeding/src/main/java/io/casehub/neocortex/memory/seeding/biography/EntityEnrichmentHandler.java` (inject CapsEngine, implement seeding)
- Modify: `memory-seeding/src/test/java/io/casehub/neocortex/memory/seeding/biography/EntityEnrichmentHandlerTest.java` (new CAPS seeding test)
- Modify: `memory-seeding/src/test/resources/biography-test/entity-enrichments.yaml` (add caps_patterns with input_activations)
- Modify: `memory-seeding/src/test/java/io/casehub/neocortex/memory/seeding/biography/BiographyImportIntegrationTest.java` (verify CAPS state)

**Interfaces:**
- Consumes: `CapsPattern.inputActivations()` from Task 1, `CapsEngine` interface from caps-api, `InMemoryCapsEngine` from caps-testing, `DispositionProfileEntry` from BiographyProfile
- Produces: `AgentCapsState` saved via `CapsEngine.saveState()` — CAPS weights seeded from biographical patterns

- [ ] **Step 1: Add Maven dependencies**

Add to `memory-seeding/pom.xml` in the `<dependencies>` section:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-caps-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognitive-index</artifactId>
</dependency>
```

And as test scope:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-caps-testing</artifactId>
    <scope>test</scope>
</dependency>
```

Verify `cognitive-index` isn't already present (it may be — `cognition-api` is listed but check if it transitively provides `DispositionAxes`).

- [ ] **Step 2: Verify dependencies resolve**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f memory-seeding/pom.xml dependency:resolve -s .mvn/slot-settings.xml`
Expected: BUILD SUCCESS

- [ ] **Step 3: Write the failing test for CAPS seeding**

Add to `EntityEnrichmentHandlerTest.java`:

```java
@Test
void seedsCapsWeightsFromInputActivations() {
    var store       = new InMemoryMindMapStore();
    var memoryStore = new InMemoryMemoryStore(principal);
    var capsEngine  = new InMemoryCapsEngine();
    var handler     = new EntityEnrichmentHandler(store, memoryStore, capsEngine, true);

    var subgraph = BiographyUtils.ensureSubgraph(store, SubgraphTypes.PERSON, TENANT);
    store.addNode(NodeInput.of("Frida Kahlo", subgraph).withProvenance("test"), TENANT);

    var patterns = List.of(
            new EntityEnrichmentEntry.CapsPattern(
                "betrayal → trust_violation → AUTONOMY_ASSERTION", 3.0, 0.87,
                Map.of("betrayal", 0.9, "abandonment", 0.6)));
    var entry = new EntityEnrichmentEntry("enrich-caps", "Frida Kahlo", null, patterns, null, null);

    var disposition = new DispositionProfileEntry(
            "disp-1", "Frida Kahlo", "cooperative", null, "bold", "high", "confrontational", null);

    var profile = new BiographyProfile("agent-1", TENANT,
            List.of(), List.of(), List.of(), List.of(), List.of(),
            List.of(), List.of(), List.of(), List.of(), List.of(),
            List.of(entry), List.of(disposition), List.of());

    handler.handle(profile, "agent-1", TENANT);

    var state = capsEngine.loadState(TENANT, "agent-1");
    assertThat(state).isNotNull();
    assertThat(state.generation()).isEqualTo(0);
    assertThat(state.weights()).isNotEmpty();
    // Verify weights differ from a fresh initialization (patterns shifted them)
    var baseline = capsEngine.initializeAgent(TENANT, "baseline-agent",
            new DispositionAxes("cooperative", null, "bold", "high", "confrontational"));
    // The betrayal→other_reliability connection should be strengthened
    var betrayalConn = "betrayal__other_reliability";
    if (state.weights().containsKey(betrayalConn) && baseline.weights().containsKey(betrayalConn)) {
        assertThat(state.weights().get(betrayalConn).effectiveWeight())
                .isNotEqualTo(baseline.weights().get(betrayalConn).effectiveWeight());
    }
}
```

Import additions needed:
```java
import io.casehub.neocortex.caps.testing.InMemoryCapsEngine;
import io.casehub.neocortex.cognitive.index.DispositionAxes;
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f memory-seeding/pom.xml test -Dtest=EntityEnrichmentHandlerTest#seedsCapsWeightsFromInputActivations -s .mvn/slot-settings.xml`
Expected: FAIL — compilation error, `EntityEnrichmentHandler` constructor doesn't accept `CapsEngine`

- [ ] **Step 5: Update EntityEnrichmentHandler — constructor and CapsEngine injection**

Add `CapsEngine` field and update constructors:

```java
import io.casehub.neocortex.caps.CapsEngine;
import io.casehub.neocortex.cognitive.index.DispositionAxes;
```

Update the class to add `CapsEngine` as an optional dependency:

```java
@ApplicationScoped
public class EntityEnrichmentHandler implements BiographyHandler {

    private final MindMapStore    store;
    private final CaseMemoryStore memoryStore;
    private final CapsEngine      capsEngine;

    @Inject
    public EntityEnrichmentHandler(MindMapStore store, CaseMemoryStore memoryStore,
                                   CapsEngine capsEngine) {
        this.store       = store;
        this.memoryStore = memoryStore;
        this.capsEngine  = capsEngine;
    }

    EntityEnrichmentHandler(MindMapStore store, CaseMemoryStore memoryStore,
                            CapsEngine capsEngine,
                            @SuppressWarnings("unused") boolean testMarker) {
        this.store       = store;
        this.memoryStore = memoryStore;
        this.capsEngine  = capsEngine;
    }
```

Keep the old 2-arg test constructor for backward compat with existing tests:

```java
    EntityEnrichmentHandler(MindMapStore store, CaseMemoryStore memoryStore,
                            @SuppressWarnings("unused") boolean testMarker) {
        this(store, memoryStore, (CapsEngine) null, true);
    }
```

- [ ] **Step 6: Implement CAPS seeding in handle() method**

Add CAPS seeding at the end of the `handle()` method, after the existing entity loop:

```java
@Override
public void handle(BiographyProfile profile, String agentId, String tenantId) {
    // --- existing entity enrichment loop (drive baselines, caps as props, attractors, stages) ---
    for (var entry : profile.entityEnrichments()) {
        // ... existing code unchanged ...
    }

    // --- CAPS weight seeding from structured input_activations ---
    seedCapsWeights(profile, agentId, tenantId);
}

private void seedCapsWeights(BiographyProfile profile, String agentId, String tenantId) {
    if (capsEngine == null) { return; }

    boolean hasActivations = profile.entityEnrichments().stream()
        .flatMap(e -> e.capsPatterns() != null ? e.capsPatterns().stream() : java.util.stream.Stream.empty())
        .anyMatch(p -> p.inputActivations() != null && !p.inputActivations().isEmpty());
    if (!hasActivations) { return; }

    DispositionAxes disposition = resolveDisposition(profile);
    var state = capsEngine.initializeAgent(tenantId, agentId, disposition);

    for (var entry : profile.entityEnrichments()) {
        if (entry.capsPatterns() == null) { continue; }
        for (var pattern : entry.capsPatterns()) {
            if (pattern.inputActivations() == null || pattern.inputActivations().isEmpty()) { continue; }
            double intensity = pattern.frequency() != null ? pattern.frequency() / 5.0 : 0.5;
            double salience = pattern.confidence() != null ? pattern.confidence() : 1.0;
            state = capsEngine.updateWeights(state, pattern.inputActivations(),
                intensity, 1.0, salience, "variable_ratio");
        }
    }

    capsEngine.saveState(state);
}

private DispositionAxes resolveDisposition(BiographyProfile profile) {
    if (profile.dispositionProfiles().isEmpty()) {
        return new DispositionAxes(null, null, null, null, null);
    }
    var dp = profile.dispositionProfiles().getFirst();
    return new DispositionAxes(
        dp.socialOrient(), dp.ruleFollowing(), dp.riskAppetite(),
        dp.autonomy(), dp.conflictMode());
}
```

- [ ] **Step 7: Run the failing test — should now pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f memory-seeding/pom.xml test -Dtest=EntityEnrichmentHandlerTest#seedsCapsWeightsFromInputActivations -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 8: Update existing tests for new constructor**

Update the existing tests that use the 2-arg test constructor — they should still work via the backward-compat constructor added in Step 5. Verify:

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f memory-seeding/pom.xml test -Dtest=EntityEnrichmentHandlerTest -s .mvn/slot-settings.xml`
Expected: PASS (all tests)

- [ ] **Step 9: Write test for skipping when no input_activations present**

Add to `EntityEnrichmentHandlerTest.java`:

```java
@Test
void skipsCapsSeedingWhenNoInputActivations() {
    var store       = new InMemoryMindMapStore();
    var memoryStore = new InMemoryMemoryStore(principal);
    var capsEngine  = new InMemoryCapsEngine();
    var handler     = new EntityEnrichmentHandler(store, memoryStore, capsEngine, true);

    var subgraph = BiographyUtils.ensureSubgraph(store, SubgraphTypes.PERSON, TENANT);
    store.addNode(NodeInput.of("Test Entity", subgraph).withProvenance("test"), TENANT);

    // Pattern with no input_activations — should NOT seed CAPS
    var patterns = List.of(
            new EntityEnrichmentEntry.CapsPattern("threat -> creative", 0.8, null, null));
    var entry = new EntityEnrichmentEntry("enrich-no-caps", "Test Entity", null, patterns, null, null);

    handler.handle(profileWith(List.of(entry)), "agent-1", TENANT);

    assertThat(capsEngine.loadState(TENANT, "agent-1")).isNull();
}
```

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f memory-seeding/pom.xml test -Dtest=EntityEnrichmentHandlerTest#skipsCapsSeedingWhenNoInputActivations -s .mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 10: Update test fixture and integration test**

Add caps_patterns with input_activations to `entity-enrichments.yaml`:

```yaml
    caps_patterns:
      - pattern: "betrayal → trust_violation → AUTONOMY_ASSERTION"
        frequency: 3
        confidence: 0.87
        input_activations:
          betrayal: 0.9
          abandonment: 0.6
```

Update `BiographyImportIntegrationTest.java`:
- Add `InMemoryCapsEngine` to the handler list
- Pass `capsEngine` to `EntityEnrichmentHandler` constructor
- Add assertion that `capsEngine.loadState(TENANT, AGENT)` returns non-null state with weights

```java
var capsEngine = new InMemoryCapsEngine();
// In handlers list, replace:
//   new EntityEnrichmentHandler(mindMapStore, memoryStore, true)
// with:
//   new EntityEnrichmentHandler(mindMapStore, memoryStore, capsEngine, true)

// After existing assertions, add:
var capsState = capsEngine.loadState(TENANT, AGENT);
assertThat(capsState).isNotNull();
assertThat(capsState.weights()).isNotEmpty();
```

- [ ] **Step 11: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f memory-seeding/pom.xml test -s .mvn/slot-settings.xml`
Expected: PASS (all tests)

- [ ] **Step 12: Commit**

```bash
git add memory-seeding/pom.xml
git add memory-seeding/src/main/java/io/casehub/neocortex/memory/seeding/biography/EntityEnrichmentHandler.java
git add memory-seeding/src/test/java/io/casehub/neocortex/memory/seeding/biography/EntityEnrichmentHandlerTest.java
git add memory-seeding/src/test/java/io/casehub/neocortex/memory/seeding/biography/BiographyImportIntegrationTest.java
git add memory-seeding/src/test/resources/biography-test/entity-enrichments.yaml
git commit -m "feat(#685): seed CAPS weights from biographical caps_patterns

EntityEnrichmentHandler now injects CapsEngine and, when structured
input_activations are present on caps_patterns, initializes the agent's
CAPS state from disposition axes then reinforces connection weights per
pattern. Parameters: frequency→intensity, confidence→salience,
variable_ratio reinforcement schedule.

CAPS settling now reflects biographical learned patterns instead of
starting from a blank slate.

Refs casehubio/neocortex#685

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [neocortex#685] — focal issue
- [caps-api/src/main/java/.../CapsEngine.java] — CapsEngine interface (initializeAgent, updateWeights, saveState)
- [caps-api/src/main/java/.../AgentCapsState.java] — CAPS state record (weights, nodeStates)
- [caps-engine/src/main/resources/caps-topology.yaml] — 30 input nodes, connections, disposition modifiers
- [caps-engine/src/main/java/.../DispositionWeightMapper.java] — disposition → weight initialization
- [caps-testing/src/main/java/.../InMemoryCapsEngine.java] — in-memory CAPS engine for tests
- [memory-seeding/src/main/java/.../EntityEnrichmentHandler.java] — handler to modify
- [memory-seeding/src/main/java/.../EntityEnrichmentEntry.java:22] — CapsPattern record to extend
- [extraction/scripts/enrich_entity.py:25-38] — DRIVES_CAPS_SYSTEM enrichment prompt
- [extraction/scripts/convert_corpus.py:350-354] — caps_patterns passthrough logic
