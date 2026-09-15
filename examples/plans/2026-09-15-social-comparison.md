# SocialComparison Integration Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #60 — SocialComparison integration
**Issue group:** #54, #55, #56, #57, #59, #60

**Goal:** Render a "Social Awareness" observation section showing
perception-level statements when a character's view of nearby characters
diverges significantly from those characters' self-perceptions.

**Architecture:** `CharacterCognition.renderSocialAwareness()` gates on
scheming/suspicion drives, calls `CognitiveProfile.compare()` per nearby
character, feeds results to `SocialComparison.compare()`, translates PAD
distance into perception statements, and returns an `ObservationSection`.
Gracefully degrades when mindmap data is absent.

**Tech Stack:** Java 26, JUnit 5, Quarkus, neocortex cognitive-index
(CognitiveProfile, SocialComparison, PerspectivalComparison)

## Global Constraints

- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
- All existing tests must pass (465+, 1 pre-existing error)
- No new dependencies — CognitiveProfile and SocialComparison already on classpath
- CognitiveProfile wired via `Instance<>` with graceful degradation

---

## Batch 1: Perception translation + Social Awareness rendering

### Task 1: PerceptionTranslator — PAD divergence to perception statements

**Files:**
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/PerceptionTranslator.java`
- Create: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/PerceptionTranslatorTest.java`

**Interfaces:**
- Consumes: `PerspectivalComparison` (neocortex) — distances, dimensionDifferences, trajectoryAlignment
- Consumes: `PrincipalId` — self and other agent identifiers
- Produces: `PerceptionTranslator.translate(PerspectivalComparison, PrincipalId self, PrincipalId other, String otherName)` → `Optional<String>`

- [ ] **Step 1: Write the failing tests**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.cognitive.index.AffectSnapshot;
import io.casehub.neocortex.cognitive.index.AffectTrajectory;
import io.casehub.neocortex.cognitive.index.PadDimension;
import io.casehub.neocortex.cognitive.index.PairwiseDifferences;
import io.casehub.neocortex.cognitive.index.PadDistanceMatrix;
import io.casehub.neocortex.cognitive.index.AgentPair;
import io.casehub.neocortex.cognitive.index.PerspectivalComparison;
import io.casehub.neocortex.cognitive.index.TrajectoryAlignment;
import io.casehub.neocortex.cognitive.index.TrendAgreement;
import io.casehub.platform.api.identity.PrincipalId;
import org.junit.jupiter.api.Test;

import java.util.Map;
import java.util.Set;

import static org.assertj.core.api.Assertions.assertThat;

class PerceptionTranslatorTest {

    private static final PrincipalId SELF = PrincipalId.of("hooded-claw");
    private static final PrincipalId OTHER = PrincipalId.of("peter-perfect");
    private static final String OTHER_NAME = "Peter Perfect";

    @Test
    void pleasureDivergenceMyLowerProducesCorrectStatement() {
        var comparison = buildComparison(0.2, 0.8, 0.5, 0.5, 0.5, 0.5);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isPresent();
        assertThat(result.get()).contains("Peter Perfect").contains("more positively");
    }

    @Test
    void pleasureDivergenceMyHigherProducesCorrectStatement() {
        var comparison = buildComparison(0.8, 0.2, 0.5, 0.5, 0.5, 0.5);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isPresent();
        assertThat(result.get()).contains("more positively about Peter Perfect");
    }

    @Test
    void arousalDivergenceProducesCorrectStatement() {
        var comparison = buildComparison(0.5, 0.5, 0.9, 0.3, 0.5, 0.5);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isPresent();
        assertThat(result.get()).contains("alert");
    }

    @Test
    void dominanceDivergenceProducesCorrectStatement() {
        var comparison = buildComparison(0.5, 0.5, 0.5, 0.5, 0.9, 0.3);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isPresent();
        assertThat(result.get()).contains("control");
    }

    @Test
    void belowThresholdReturnsEmpty() {
        var comparison = buildComparison(0.5, 0.6, 0.5, 0.5, 0.5, 0.5);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isEmpty();
    }

    @Test
    void unassessedAgentReturnsEmpty() {
        var pair = AgentPair.of(SELF, OTHER);
        var comparison = new PerspectivalComparison(
                "peter-node", OTHER_NAME,
                Map.of(SELF, new AffectSnapshot(SELF, null, null, null, null),
                       OTHER, new AffectSnapshot(OTHER, 0.5, 0.5, 0.5, null)),
                Set.of(SELF),
                new PadDistanceMatrix(Map.of()),
                Map.of(),
                new TrajectoryAlignment(Map.of(), Map.of()),
                2);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isEmpty();
    }

    @Test
    void divergentTrajectoryAppendsSuffix() {
        var pair = AgentPair.of(SELF, OTHER);
        var snapshots = Map.of(
                SELF, new AffectSnapshot(SELF, 0.2, 0.5, 0.5, null),
                OTHER, new AffectSnapshot(OTHER, 0.8, 0.5, 0.5, null));
        var distances = new PadDistanceMatrix(Map.of(pair, 0.6));
        var diffs = Map.of(
                PadDimension.PLEASURE, new PairwiseDifferences(Map.of(pair, -0.6)),
                PadDimension.AROUSAL, new PairwiseDifferences(Map.of(pair, 0.0)),
                PadDimension.DOMINANCE, new PairwiseDifferences(Map.of(pair, 0.0)));
        var alignment = new TrajectoryAlignment(
                Map.of(pair, -0.5),
                Map.of(pair, TrendAgreement.DIVERGENT));
        var comparison = new PerspectivalComparison(
                "peter-node", OTHER_NAME, snapshots, Set.of(),
                distances, diffs, alignment, 2);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isPresent();
        assertThat(result.get()).contains("widening");
    }

    @Test
    void alignedTrajectoryAppendsSuffix() {
        var pair = AgentPair.of(SELF, OTHER);
        var snapshots = Map.of(
                SELF, new AffectSnapshot(SELF, 0.2, 0.5, 0.5, null),
                OTHER, new AffectSnapshot(OTHER, 0.8, 0.5, 0.5, null));
        var distances = new PadDistanceMatrix(Map.of(pair, 0.6));
        var diffs = Map.of(
                PadDimension.PLEASURE, new PairwiseDifferences(Map.of(pair, -0.6)),
                PadDimension.AROUSAL, new PairwiseDifferences(Map.of(pair, 0.0)),
                PadDimension.DOMINANCE, new PairwiseDifferences(Map.of(pair, 0.0)));
        var alignment = new TrajectoryAlignment(
                Map.of(pair, 0.8),
                Map.of(pair, TrendAgreement.ALIGNED));
        var comparison = new PerspectivalComparison(
                "peter-node", OTHER_NAME, snapshots, Set.of(),
                distances, diffs, alignment, 2);
        var result = PerceptionTranslator.translate(comparison, SELF, OTHER, OTHER_NAME);
        assertThat(result).isPresent();
        assertThat(result.get()).contains("converging");
    }

    private PerspectivalComparison buildComparison(
            double selfPleasure, double otherPleasure,
            double selfArousal, double otherArousal,
            double selfDominance, double otherDominance) {
        var pair = AgentPair.of(SELF, OTHER);
        var snapshots = Map.of(
                SELF, new AffectSnapshot(SELF, selfPleasure, selfArousal, selfDominance, null),
                OTHER, new AffectSnapshot(OTHER, otherPleasure, otherArousal, otherDominance, null));
        double dp = selfPleasure - otherPleasure;
        double da = selfArousal - otherArousal;
        double dd = selfDominance - otherDominance;
        double distance = Math.sqrt(dp * dp + da * da + dd * dd);
        var distances = new PadDistanceMatrix(Map.of(pair, distance));
        var diffs = Map.of(
                PadDimension.PLEASURE, new PairwiseDifferences(Map.of(pair, dp)),
                PadDimension.AROUSAL, new PairwiseDifferences(Map.of(pair, da)),
                PadDimension.DOMINANCE, new PairwiseDifferences(Map.of(pair, dd)));
        var alignment = new TrajectoryAlignment(Map.of(), Map.of());
        return new PerspectivalComparison(
                "other-node", OTHER_NAME, snapshots, Set.of(),
                distances, diffs, alignment, 2);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=PerceptionTranslatorTest`
Expected: compilation failure — `PerceptionTranslator` does not exist

- [ ] **Step 3: Implement PerceptionTranslator**

```java
package io.casehub.examples.manor.agent;

import io.casehub.neocortex.cognitive.index.AgentPair;
import io.casehub.neocortex.cognitive.index.PadDimension;
import io.casehub.neocortex.cognitive.index.PerspectivalComparison;
import io.casehub.neocortex.cognitive.index.TrendAgreement;
import io.casehub.platform.api.identity.PrincipalId;

import java.util.Optional;

public final class PerceptionTranslator {

    static final double DISTANCE_THRESHOLD = 0.4;

    private PerceptionTranslator() {}

    public static Optional<String> translate(
            PerspectivalComparison comparison,
            PrincipalId self,
            PrincipalId other,
            String otherName) {

        if (comparison.unassessedAgents().contains(self)
                || comparison.unassessedAgents().contains(other)) {
            return Optional.empty();
        }

        var pair = AgentPair.of(self, other);
        double distance = comparison.distances().distance(self, other);
        if (distance < DISTANCE_THRESHOLD) {
            return Optional.empty();
        }

        PadDimension dominant = dominantDimension(comparison, pair);
        double diff = comparison.dimensionDifferences().get(dominant)
                .differences().getOrDefault(pair, 0.0);

        String statement = switch (dominant) {
            case PLEASURE -> diff < 0
                    ? otherName + " seems to view this more positively than you do"
                    : "You feel more positively about " + otherName + " than they feel about themselves";
            case AROUSAL -> diff > 0
                    ? "You're more alert around " + otherName + " than they seem to be"
                    : otherName + " seems more on edge than you'd expect";
            case DOMINANCE -> diff > 0
                    ? "You feel more in control around " + otherName + " than they do"
                    : otherName + " seems more confident in this interaction than you are";
        };

        var trajectory = comparison.trajectoryAlignment();
        var agreement = trajectory.agreements().get(pair);
        if (agreement == TrendAgreement.DIVERGENT) {
            statement += " (and this gap is widening)";
        } else if (agreement == TrendAgreement.ALIGNED) {
            statement += " (though you're converging)";
        }

        return Optional.of(statement);
    }

    private static PadDimension dominantDimension(
            PerspectivalComparison comparison, AgentPair pair) {
        PadDimension dominant = PadDimension.PLEASURE;
        double maxAbs = 0;
        for (PadDimension dim : PadDimension.values()) {
            var diffs = comparison.dimensionDifferences().get(dim);
            if (diffs != null) {
                double abs = Math.abs(diffs.differences().getOrDefault(pair, 0.0));
                if (abs > maxAbs) {
                    maxAbs = abs;
                    dominant = dim;
                }
            }
        }
        return dominant;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=PerceptionTranslatorTest`
Expected: all 8 tests PASS

- [ ] **Step 5: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/PerceptionTranslator.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/PerceptionTranslatorTest.java
git commit -m "feat(#60): PerceptionTranslator — PAD divergence to perception-level statements

Translates PerspectivalComparison PAD distances into character-perspective
observations. Gates on distance threshold (0.4), selects dominant PAD
dimension, appends trajectory suffix when available.

Refs #60

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: CharacterCognition.renderSocialAwareness() + wiring

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java:94-141`
- Modify: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java`

**Interfaces:**
- Consumes: `PerceptionTranslator.translate(PerspectivalComparison, PrincipalId, PrincipalId, String)` from Task 1
- Consumes: `CognitiveProfile.compare(CognitiveProfileQuery, Set<PrincipalId>)` — neocortex API
- Consumes: `SocialComparison.compare(Map<PrincipalId, EntityKnowledge>)` — neocortex API
- Produces: `CharacterCognition.renderSocialAwareness(Collection<String> nearbyAgentIds, Map<String, String> agentNames)` → `Optional<ObservationSection>`

- [ ] **Step 1: Write the failing tests**

Add tests to `CharacterCognitionTest`:

```java
@Test
void socialAwarenessAbsentWithoutSchemingDrives() {
    var socialConfig = ManorSocialConfigLoader.load().get("penelope-pitstop");
    var cognition = new CharacterCognition("penelope-pitstop", null, null, socialConfig, List.of());
    var sections = cognition.renderCognitiveSections(
            new CharacterState("penelope-pitstop", "Penelope", "Room", 0.0, List.of()),
            List.of("hooded-claw"), Map.of("hooded-claw", "Hooded Claw"));
    assertThat(sections.stream().map(s -> s.header()).toList())
            .doesNotContain("Social Awareness");
}

@Test
void socialAwarenessAbsentWhenCognitiveProfileNull() {
    var drives = List.of(new SocialConfig.Drive("scheming", 0.9, "Schemes"));
    var socialConfig = new SocialConfig(drives, List.of(), List.of());
    var cognition = new CharacterCognition("hooded-claw", null, null, socialConfig, List.of());
    var sections = cognition.renderCognitiveSections(
            new CharacterState("hooded-claw", "HC", "Room", 0.0, List.of()),
            List.of("peter-perfect"), Map.of("peter-perfect", "Peter Perfect"));
    assertThat(sections.stream().map(s -> s.header()).toList())
            .doesNotContain("Social Awareness");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest="CharacterCognitionTest#socialAwarenessAbsentWithoutSchemingDrives+socialAwarenessAbsentWhenCognitiveProfileNull"`
Expected: compilation failure — method doesn't exist yet (but these tests check absence, so they may actually pass if the section simply isn't rendered. Let them compile first.)

- [ ] **Step 3: Implement renderSocialAwareness() in CharacterCognition**

Add to `CharacterCognition.java` after `renderCognitiveSections()`:

```java
private static final double SOCIAL_AWARENESS_DRIVE_THRESHOLD = 0.5;
private static final Set<String> SOCIAL_AWARENESS_DRIVES = Set.of("scheming", "suspicion");

List<ObservationSection> renderSocialAwareness(
        Collection<String> nearbyAgentIds,
        Map<String, String> agentNames) {
    if (cognitiveProfile == null || tenantId == null) {
        return List.of();
    }
    boolean hasSocialDrive = socialConfig.drives().stream()
            .anyMatch(d -> SOCIAL_AWARENESS_DRIVES.contains(d.type())
                    && d.intensity() > SOCIAL_AWARENESS_DRIVE_THRESHOLD);
    if (!hasSocialDrive) {
        return List.of();
    }

    var selfPrincipal = io.casehub.platform.api.identity.PrincipalId.of(agentId);
    var lines = new java.util.ArrayList<String>();

    for (String nearbyId : nearbyAgentIds) {
        var otherPrincipal = io.casehub.platform.api.identity.PrincipalId.of(nearbyId);
        String otherName = agentNames.getOrDefault(nearbyId, nearbyId);
        try {
            var query = io.casehub.neocortex.cognitive.index.CognitiveProfileQuery
                    .byName(nearbyId, tenantId);
            var perspectives = cognitiveProfile.compare(query,
                    java.util.Set.of(selfPrincipal, otherPrincipal));
            if (perspectives.isEmpty()) continue;

            var comparison = io.casehub.neocortex.cognitive.index.SocialComparison
                    .compare(perspectives);
            PerceptionTranslator.translate(comparison, selfPrincipal, otherPrincipal, otherName)
                    .ifPresent(lines::add);
        } catch (Exception e) {
            // graceful degradation — no section if compare fails
        }
    }

    if (lines.isEmpty()) return List.of();
    return List.of(ObservationSection.items("Social Awareness", null, lines));
}
```

- [ ] **Step 4: Wire renderSocialAwareness into renderCognitiveSections**

In `renderCognitiveSections()`, after the CognitionCore block (after line ~140), add:

```java
sections.addAll(renderSocialAwareness(nearbyAgentIds, agentNames));
```

- [ ] **Step 5: Run all tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS (465+, 1 pre-existing error only)

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/CharacterCognitionTest.java
git commit -m "feat(#60): renderSocialAwareness() — perception divergence observation section

Drive-gated (scheming/suspicion > 0.5) Social Awareness section.
Calls CognitiveProfile.compare() per nearby character, feeds to
SocialComparison, renders perception-level statements via
PerceptionTranslator. Gracefully degrades when mindmap unavailable.

Refs #60

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-15-social-comparison-design.md] — design spec this plan implements
- CognitiveProfile.compare() — neocortex, `Map<PrincipalId, EntityKnowledge>` return type
- CognitiveProfileQuery.byName(entityName, tenantId) — query factory
- SocialComparison.compare(perspectives) — returns PerspectivalComparison
- PerspectivalComparison — distances, dimensionDifferences, trajectoryAlignment
- PadDistanceMatrix.distance(PrincipalId, PrincipalId) — pairwise distance
- PairwiseDifferences — per-dimension differences
- TrajectoryAlignment — agreements (ALIGNED, DIVERGENT, MIXED, INSUFFICIENT)
- AffectSnapshot — per-agent PAD values
- CharacterCognition.java:94-141 — renderCognitiveSections
- social-config.yaml — drive definitions
- GE-20260915-2b5e80 — comparative data perception-level translation
- GitHub #60 — SocialComparison integration
