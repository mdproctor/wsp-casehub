# SocialComparison Integration — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #317 — SocialComparison integration
**Issue group:** #317

**Goal:** Wire neocortex SocialComparison into blocks so agents surface multi-agent perspectival divergence (PAD distances, pairwise differences, trajectory alignment) in prompts.

**Architecture:** Extend `CognitiveProfileParticipant` to run `SocialComparison.compare()` after `CognitiveProfile.compare()`, caching `PerspectivalComparison` per entity. New `SocialComparisonPromptSection` renders divergence metrics as structured NL. Remove superseded comparison rendering from `EntityKnowledgePromptSection`.

**Tech Stack:** Java 21, Quarkus CDI, Spring Boot, casehub-neocortex-cognitive-index (SocialComparison, PerspectivalComparison)

## Global Constraints

- Pre-release stage — no backward compatibility required
- Gated by existing `CognitionConfig.perspectiveComparisonEnabled()` — no new config flags
- Graceful degradation — when CognitiveProfile not on classpath, nothing renders
- Use `ide_insert_member`, `ide_replace_member` for structural edits
- Use `ide_refactor_rename` for renames
- `mvn test -pl blocks-core` and `mvn test -pl blocks` for verification
- Slot settings: `-s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml`

---

## Batch 1: Extend Participant + New Section

### Task 1: Extend CognitiveProfileParticipant — add SocialComparison step

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitiveProfileParticipant.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/CognitiveProfileParticipantTest.java`

**Interfaces:**
- Consumes: `SocialComparison.compare(Map<PrincipalId, EntityKnowledge>) → PerspectivalComparison`, existing `lastComparisons()` cache
- Produces: `CognitiveProfileParticipant.lastSocialComparisons() → Map<String, PerspectivalComparison>` (keyed by entity node ID)

- [ ] **Step 1: Write failing test**

```java
@Test
void tickPopulatesSocialComparisonsWhenPerspectiveComparisonEnabled() {
    var profile = mock(CognitiveProfile.class);
    var node = stubNode("penelope");
    var entity = stubEntityKnowledge(node);
    when(profile.resolve(any())).thenReturn(Optional.of(entity));
    when(profile.compare(any(), any())).thenReturn(Map.of(
            PrincipalId.agent("agent-a"), entity,
            PrincipalId.agent("agent-b"), entity));

    var config = CognitionConfig.all().with("perspectiveComparison", true);
    var participant = new CognitiveProfileParticipant(profile, null, null, config);

    var context = new CognitionTickContext("agent-a", "tenant",
            null, (aid, tid) -> Set.of("penelope"));
    participant.tick(context);

    assertThat(participant.lastSocialComparisons()).hasSize(1);
    assertThat(participant.lastSocialComparisons().values().iterator().next()
            .agentCount()).isEqualTo(2);
}

@Test
void tickSocialComparisonsEmptyWhenPerspectiveComparisonDisabled() {
    var profile = mock(CognitiveProfile.class);
    var node = stubNode("penelope");
    var entity = stubEntityKnowledge(node);
    when(profile.resolve(any())).thenReturn(Optional.of(entity));

    var participant = new CognitiveProfileParticipant(
            profile, null, null, CognitionConfig.all());

    var context = new CognitionTickContext("agent", "tenant",
            null, (aid, tid) -> Set.of("penelope"));
    participant.tick(context);

    assertThat(participant.lastSocialComparisons()).isEmpty();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -Dtest=CognitiveProfileParticipantTest -f /Users/mdproctor/claude/casehub/slots/196/blocks/pom.xml -s /Users/mdproctor/claude/casehub/slots/196/examples/.mvn/slot-settings.xml`
Expected: compilation failure — `lastSocialComparisons()` not defined

- [ ] **Step 3: Add SocialComparison step to CognitiveProfileParticipant**

Add new field:
```java
private Map<String, PerspectivalComparison> lastSocialComparisons = Map.of();
```

In `tick()`, after existing `comparisons.put(ek.node().id(), cmp);`, add:
```java
lastSocialComparisons.put(ek.node().id(),
        SocialComparison.compare(cmp));
```

Initialize `lastSocialComparisons` to empty at top of `tick()`:
```java
var socialComparisons = new LinkedHashMap<String, PerspectivalComparison>();
```

At end of tick:
```java
this.lastSocialComparisons = Map.copyOf(socialComparisons);
```

Add accessor:
```java
public Map<String, PerspectivalComparison> lastSocialComparisons() {
    return lastSocialComparisons;
}
```

Add imports:
```java
import io.casehub.neocortex.cognitive.index.PerspectivalComparison;
import io.casehub.neocortex.cognitive.index.SocialComparison;
```

- [ ] **Step 4: Install blocks-core and run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl blocks-core -DskipTests -f ... -s ...`
Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -Dtest=CognitiveProfileParticipantTest -f ... -s ...`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/src/main/java/io/casehub/blocks/agentic/social/CognitiveProfileParticipant.java blocks/src/test/java/io/casehub/blocks/agentic/social/CognitiveProfileParticipantTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(casehubio/blocks#317): extend CognitiveProfileParticipant with SocialComparison step"
```

### Task 2: SocialComparisonPromptSection — render perspectival divergence

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/SocialComparisonPromptSection.java`
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/SocialComparisonPromptSectionTest.java`

**Interfaces:**
- Consumes: `CognitiveProfileParticipant.lastSocialComparisons() → Map<String, PerspectivalComparison>`, `PerspectivalComparison(entityId, entityName, perspectives, unassessedAgents, distances, dimensionDifferences, trajectoryAlignment, agentCount)`
- Produces: `SocialComparisonPromptSection implements PromptSection` — `contribute(PromptContext) → @Nullable String`

- [ ] **Step 1: Write failing test — divergent pair renders**

```java
@Test
void rendersDivergentPair() {
    var agentA = PrincipalId.agent("hooded-claw");
    var agentB = PrincipalId.agent("muttley");

    var snapshots = Map.of(
            agentA, new AffectSnapshot(agentA, 0.8, 0.3, 0.5, null),
            agentB, new AffectSnapshot(agentB, 0.2, 0.7, 0.3, null));

    var pair = AgentPair.of(agentA, agentB);
    var distances = new PadDistanceMatrix(Map.of(pair, 0.73));
    var diffs = Map.of(
            PadDimension.PLEASURE, new PairwiseDifferences(Map.of(pair, 0.6)),
            PadDimension.AROUSAL, new PairwiseDifferences(Map.of(pair, -0.4)),
            PadDimension.DOMINANCE, new PairwiseDifferences(Map.of(pair, 0.2)));
    var alignment = new TrajectoryAlignment(Map.of(pair, -0.5),
            Map.of(pair, TrendAgreement.DIVERGENT));

    var comparison = new PerspectivalComparison("node-1", "Penelope",
            snapshots, Set.of(), distances, diffs, alignment, 2);

    var section = new SocialComparisonPromptSection(
            Map.of("node-1", comparison), 0.4);
    var text = section.contribute(new PromptContext("hooded-claw", "tenant", null));

    assertThat(text).contains("Penelope");
    assertThat(text).contains("0.73");
    assertThat(text).contains("pleasure");
    assertThat(text).contains("DIVERGENT");
}
```

- [ ] **Step 2: Write failing test — aligned pair filtered out**

```java
@Test
void filtersAlignedPairsBelowThreshold() {
    var agentA = PrincipalId.agent("a");
    var agentB = PrincipalId.agent("b");
    var pair = AgentPair.of(agentA, agentB);
    var snapshots = Map.of(
            agentA, new AffectSnapshot(agentA, 0.5, 0.5, 0.5, null),
            agentB, new AffectSnapshot(agentB, 0.5, 0.5, 0.5, null));
    var distances = new PadDistanceMatrix(Map.of(pair, 0.1));
    var diffs = Map.of(
            PadDimension.PLEASURE, new PairwiseDifferences(Map.of(pair, 0.0)),
            PadDimension.AROUSAL, new PairwiseDifferences(Map.of(pair, 0.0)),
            PadDimension.DOMINANCE, new PairwiseDifferences(Map.of(pair, 0.0)));
    var alignment = new TrajectoryAlignment(Map.of(pair, 1.0),
            Map.of(pair, TrendAgreement.ALIGNED));

    var comparison = new PerspectivalComparison("node-1", "Penelope",
            snapshots, Set.of(), distances, diffs, alignment, 2);

    var section = new SocialComparisonPromptSection(
            Map.of("node-1", comparison), 0.4);
    assertThat(section.contribute(new PromptContext("a", "t", null))).isNull();
}
```

- [ ] **Step 3: Run tests — verify they fail**

Expected: compilation failure — `SocialComparisonPromptSection` does not exist

- [ ] **Step 4: Implement SocialComparisonPromptSection**

```java
package io.casehub.blocks.agentic.social.prompt;

import io.casehub.blocks.speech.PromptContext;
import io.casehub.blocks.speech.PromptSection;
import io.casehub.neocortex.cognitive.index.AgentPair;
import io.casehub.neocortex.cognitive.index.PadDimension;
import io.casehub.neocortex.cognitive.index.PerspectivalComparison;
import io.casehub.neocortex.cognitive.index.TrendAgreement;
import org.jspecify.annotations.Nullable;

import java.util.Map;

public class SocialComparisonPromptSection implements PromptSection {

    private static final int MAX_PAIRS_PER_ENTITY = 5;
    private static final double DEFAULT_DISTANCE_THRESHOLD = 0.4;

    private final Map<String, PerspectivalComparison> comparisons;
    private final double distanceThreshold;

    public SocialComparisonPromptSection(Map<String, PerspectivalComparison> comparisons) {
        this(comparisons, DEFAULT_DISTANCE_THRESHOLD);
    }

    public SocialComparisonPromptSection(Map<String, PerspectivalComparison> comparisons,
                                          double distanceThreshold) {
        this.comparisons = comparisons;
        this.distanceThreshold = distanceThreshold;
    }

    @Override
    public @Nullable String contribute(PromptContext context) {
        if (comparisons.isEmpty()) return null;
        var sb = new StringBuilder();
        for (var entry : comparisons.entrySet()) {
            renderEntity(sb, entry.getValue());
        }
        return sb.isEmpty() ? null : sb.toString().strip();
    }

    private void renderEntity(StringBuilder sb, PerspectivalComparison comparison) {
        var distances = comparison.distances().distances();
        var rendered = 0;
        var entityBlock = new StringBuilder();

        for (var distEntry : distances.entrySet()) {
            if (rendered >= MAX_PAIRS_PER_ENTITY) break;
            var pair = distEntry.getKey();
            var distance = distEntry.getValue();
            if (distance <= distanceThreshold) continue;

            var agreement = comparison.trajectoryAlignment().agreements().get(pair);
            if (agreement == TrendAgreement.ALIGNED) continue;

            if (entityBlock.isEmpty()) {
                entityBlock.append("Social divergence — ").append(comparison.entityName()).append(":");
            }

            entityBlock.append("\n  ").append(pair.a().value())
                       .append(" ↔ ").append(pair.b().value())
                       .append(": distance ").append(String.format("%.2f", distance));

            var dominant = dominantDimension(comparison, pair);
            if (dominant != null) {
                var diff = comparison.dimensionDifferences().get(dominant.dim)
                                     .differences().get(pair);
                entityBlock.append("\n    Dominant difference: ")
                           .append(dominant.dim.name().toLowerCase())
                           .append(" — ").append(pair.a().value())
                           .append(diff > 0 ? " +" : " ")
                           .append(String.format("%.2f", diff))
                           .append(" vs ").append(pair.b().value());
            }

            if (agreement != null && agreement != TrendAgreement.INSUFFICIENT) {
                entityBlock.append("\n    Trajectory: ").append(agreement.name());
            }
            rendered++;
        }

        if (!entityBlock.isEmpty()) {
            if (!sb.isEmpty()) sb.append("\n\n");
            sb.append(entityBlock);
        }
    }

    private record DimDiff(PadDimension dim, double absDiff) {}

    private @Nullable DimDiff dominantDimension(PerspectivalComparison comparison,
                                                  AgentPair pair) {
        DimDiff max = null;
        for (var dimEntry : comparison.dimensionDifferences().entrySet()) {
            var diff = dimEntry.getValue().differences().get(pair);
            if (diff != null) {
                var abs = Math.abs(diff);
                if (max == null || abs > max.absDiff) {
                    max = new DimDiff(dimEntry.getKey(), abs);
                }
            }
        }
        return max;
    }
}
```

- [ ] **Step 5: Install blocks-core and run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl blocks-core -DskipTests -f ... -s ...`
Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -Dtest=SocialComparisonPromptSectionTest -f ... -s ...`
Expected: all tests pass

- [ ] **Step 6: Write additional tests**

```java
@Test
void returnsNullWhenEmpty() {
    var section = new SocialComparisonPromptSection(Map.of());
    assertThat(section.contribute(new PromptContext("a", "t", null))).isNull();
}

@Test
void rendersMixedTrajectory() {
    // Set up pair with TrendAgreement.MIXED, distance > 0.4
    // Verify text contains "MIXED"
}

@Test
void respectsMaxPairsPerEntityLimit() {
    // Create comparison with 7 divergent pairs
    // Verify only 5 rendered
}
```

- [ ] **Step 7: Run all blocks-core + blocks tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -f ... -s ...`
Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -f ... -s ...`
Expected: all pass

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/SocialComparisonPromptSection.java blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/SocialComparisonPromptSectionTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(casehubio/blocks#317): add SocialComparisonPromptSection — perspectival divergence rendering"
```

---

## Batch 2: Wiring + Cleanup

### Task 3: Wire SocialComparisonPromptSection + remove EntityKnowledgePromptSection comparisons

**Files:**
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/SocialAvatarCognition.java` (section customizer)
- Modify: `blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/EntityKnowledgePromptSection.java` (remove comparison)
- Test: `blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/EntityKnowledgePromptSectionTest.java` (update)

**Interfaces:**
- Consumes: `CognitiveProfileParticipant.lastSocialComparisons()`, `SocialComparisonPromptSection` constructor
- Produces: Social comparison section appears in `CognitionCore.promptSections()` output

- [ ] **Step 1: Update SocialAvatarCognition section customizer**

In the existing CognitiveProfile wiring block, update the `chainSectionCustomizer` lambda to also add `SocialComparisonPromptSection`:

Before (current code):
```java
core.chainSectionCustomizer(sections -> {
    var result = new java.util.ArrayList<>(sections);
    var knowledge = profileParticipant.lastEntityKnowledge();
    if (!knowledge.isEmpty()) {
        result.add(new EntityKnowledgePromptSection(
                knowledge, profileParticipant.lastComparisons()));
    }
    return result;
});
```

After:
```java
core.chainSectionCustomizer(sections -> {
    var result = new java.util.ArrayList<>(sections);
    var knowledge = profileParticipant.lastEntityKnowledge();
    if (!knowledge.isEmpty()) {
        result.add(new EntityKnowledgePromptSection(knowledge));
    }
    var socialComparisons = profileParticipant.lastSocialComparisons();
    if (!socialComparisons.isEmpty()) {
        result.add(new SocialComparisonPromptSection(socialComparisons));
    }
    return result;
});
```

Note: `EntityKnowledgePromptSection` constructor changes — no longer takes comparisons.

- [ ] **Step 2: Remove comparison rendering from EntityKnowledgePromptSection**

Remove:
- `comparisons` field and constructor parameter
- `renderComparison()` method
- Comparison invocation in `contribute()`

Update constructor to:
```java
public EntityKnowledgePromptSection(List<EntityKnowledge> entities) {
    this.entities = entities;
}
```

Remove from `contribute()`:
```java
var cmp = comparisons.get(ek.node().id());
if (cmp != null && !cmp.isEmpty()) renderComparison(sb, ek, cmp);
```

- [ ] **Step 3: Update EntityKnowledgePromptSectionTest**

Remove comparison-related tests (`rendersComparisonWhenPresent`).
Update all `new EntityKnowledgePromptSection(List.of(ek), Map.of())` calls to `new EntityKnowledgePromptSection(List.of(ek))`.

- [ ] **Step 4: Install blocks-core and run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn install -pl blocks-core -DskipTests -f ... -s ...`
Then: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core,blocks -f ... -s ...`
Expected: all tests pass

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/SocialAvatarCognition.java blocks-core/src/main/java/io/casehub/blocks/agentic/social/prompt/EntityKnowledgePromptSection.java blocks/src/test/java/io/casehub/blocks/agentic/social/prompt/EntityKnowledgePromptSectionTest.java
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(casehubio/blocks#317): wire SocialComparisonPromptSection + remove duplicate comparison rendering"
```

---

## References

- [2026-09-29-social-comparison-integration-design.md] — design spec this plan implements
- [SocialComparison.java] — neocortex cognitive-index, static compare() utility
- [PerspectivalComparison.java] — record: perspectives, distances, dimensionDifferences, trajectoryAlignment
- [CognitiveProfileParticipant.java] — blocks-core, TERMINAL tick participant (#319)
- [EntityKnowledgePromptSection.java] — blocks-core, current comparison rendering to be removed
- [SocialAvatarCognition.java] — blocks-core, Builder + section customizer wiring
- [AttentionPromptSection.java] — blocks-core, prompt section pattern reference
- [PerceptionTranslator.java] — wacky-manor, 0.4 distance threshold source
- [GitHub #317] — SocialComparison integration
- [GitHub #319] — CognitiveProfile deep integration (foundation)
- [GitHub #311] — Neocortex cognitive integration epic
