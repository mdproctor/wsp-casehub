# Social Cognition Layer Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #52 — Wacky Manor Phase A: social cognition layer
**Issue group:** #52

**Goal:** Integrate blocks social cognition into wacky-manor — belief revision, social drives, normative reasoning, and trust/disposition subsystem replacements — so characters form opinions, follow personal codes, and pursue social agendas.

**Architecture:** Hybrid integration. Replace ManorTrustProvider with per-relationship ManorRelationshipTrust. Replace ManorDispositionRecorder with blocks memory scoring. Layer beliefs, drives, and norms as new prompt-visible cognitive sections via ObservationBuilder. All social config lives in Eidos descriptors via extensionData.

**Tech Stack:** Java 21, Quarkus, Eidos API (extensionData), blocks (memory scoring, normative, social.drive, agentic.belief, social.prompt), ObservationPipeline

## Global Constraints

- Social cognition is prompt-visible — injected as observation sections the LLM sees
- Belief revision is rule-based, not LLM-driven — deterministic and fast
- Drives are static per scenario — loaded once from Eidos descriptors
- No changes to goal/plan/reflection systems — keep existing ManorGoal*, ManorPlan*
- All new types in `io.casehub.examples.manor.agent` package
- TDD: failing test first, then minimal implementation

---

## Batch 1: Social cognition data model

### Task 1: SocialCognitionLoader — parse social config from Eidos descriptors

Loads the `social` section from each character's Eidos descriptor `extensionData` map. Caches per scenario.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/SocialCognitionLoader.java`
- Create: `src/main/java/io/casehub/examples/manor/agent/SocialConfig.java`
- Test: `src/test/java/io/casehub/examples/manor/agent/SocialCognitionLoaderTest.java`

**Interfaces:**
- Consumes: `AgentRegistry.findById(agentId, tenancyId)` → `AgentDescriptor.extensionData()`
- Produces: `SocialConfig load(String agentId)` returning drives, norms, initial beliefs

- [ ] **Step 1: Write the data records**

```java
package io.casehub.examples.manor.agent;

import java.util.List;
import java.util.Map;

public record SocialConfig(
    List<DriveEntry> drives,
    List<NormEntry> norms,
    List<BeliefEntry> initialBeliefs
) {
    public static final SocialConfig EMPTY = new SocialConfig(List.of(), List.of(), List.of());

    public record DriveEntry(String type, double intensity, String description) {}
    public record NormEntry(String rule, int priority) {}
    public record BeliefEntry(String key, String value) {}
}
```

- [ ] **Step 2: Write failing test**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class SocialCognitionLoaderTest {

    @Test
    void parsesExtensionDataWithAllSections() {
        Map<String, Object> ext = Map.of("social", Map.of(
            "drives", List.of(
                Map.of("type", "scheming", "intensity", 0.9, "description", "Compelled to scheme")
            ),
            "norms", List.of(
                Map.of("rule", "Never help Penelope", "priority", 10)
            ),
            "initial-beliefs", List.of(
                Map.of("key", "penelope-awareness", "value", "Penelope is naive")
            )
        ));
        var config = SocialCognitionLoader.parseExtensionData(ext);
        assertThat(config.drives()).hasSize(1);
        assertThat(config.drives().get(0).type()).isEqualTo("scheming");
        assertThat(config.drives().get(0).intensity()).isEqualTo(0.9);
        assertThat(config.norms()).hasSize(1);
        assertThat(config.norms().get(0).rule()).isEqualTo("Never help Penelope");
        assertThat(config.initialBeliefs()).hasSize(1);
        assertThat(config.initialBeliefs().get(0).key()).isEqualTo("penelope-awareness");
    }

    @Test
    void returnsEmptyWhenNoSocialSection() {
        var config = SocialCognitionLoader.parseExtensionData(Map.of());
        assertThat(config).isEqualTo(SocialConfig.EMPTY);
    }

    @Test
    void handlesPartialSocialSection() {
        Map<String, Object> ext = Map.of("social", Map.of(
            "drives", List.of(
                Map.of("type", "curiosity", "intensity", 0.5, "description", "Curious")
            )
        ));
        var config = SocialCognitionLoader.parseExtensionData(ext);
        assertThat(config.drives()).hasSize(1);
        assertThat(config.norms()).isEmpty();
        assertThat(config.initialBeliefs()).isEmpty();
    }
}
```

- [ ] **Step 3: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=SocialCognitionLoaderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — SocialCognitionLoader doesn't exist yet.

- [ ] **Step 4: Implement SocialCognitionLoader**

```java
package io.casehub.examples.manor.agent;

import java.util.List;
import java.util.Map;

public final class SocialCognitionLoader {

    @SuppressWarnings("unchecked")
    public static SocialConfig parseExtensionData(Map<String, Object> extensionData) {
        var social = (Map<String, Object>) extensionData.get("social");
        if (social == null) return SocialConfig.EMPTY;

        var drives = parseDrives((List<Map<String, Object>>) social.getOrDefault("drives", List.of()));
        var norms = parseNorms((List<Map<String, Object>>) social.getOrDefault("norms", List.of()));
        var beliefs = parseBeliefs((List<Map<String, Object>>) social.getOrDefault("initial-beliefs", List.of()));
        return new SocialConfig(drives, norms, beliefs);
    }

    private static List<SocialConfig.DriveEntry> parseDrives(List<Map<String, Object>> raw) {
        return raw.stream()
            .map(m -> new SocialConfig.DriveEntry(
                (String) m.get("type"),
                ((Number) m.get("intensity")).doubleValue(),
                (String) m.get("description")))
            .toList();
    }

    private static List<SocialConfig.NormEntry> parseNorms(List<Map<String, Object>> raw) {
        return raw.stream()
            .map(m -> new SocialConfig.NormEntry(
                (String) m.get("rule"),
                ((Number) m.get("priority")).intValue()))
            .toList();
    }

    private static List<SocialConfig.BeliefEntry> parseBeliefs(List<Map<String, Object>> raw) {
        return raw.stream()
            .map(m -> new SocialConfig.BeliefEntry(
                (String) m.get("key"),
                (String) m.get("value")))
            .toList();
    }
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=SocialCognitionLoaderTest`
Expected: all 3 tests PASS.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialCognitionLoader.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/SocialCognitionLoaderTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): SocialCognitionLoader — parse social config from Eidos extensionData"
```

---

### Task 2: BeliefStore — per-character beliefs with revision tracking

In-memory belief map per character. Tracks revision history for prompt rendering.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/BeliefStore.java`
- Test: `src/test/java/io/casehub/examples/manor/agent/BeliefStoreTest.java`

**Interfaces:**
- Consumes: `SocialConfig.initialBeliefs()` from Task 1
- Produces: `BeliefStore` with `init(String agentId, List<BeliefEntry>)`, `get(String agentId)`, `revise(String agentId, String key, String newValue, int tick)`, `beliefs(String agentId)` returning `List<StoredBelief>`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class BeliefStoreTest {

    @Test
    void initAndRetrieve() {
        var store = new BeliefStore();
        store.init("hooded-claw", List.of(
            new SocialConfig.BeliefEntry("poison-location", "Library")
        ));
        var beliefs = store.beliefs("hooded-claw");
        assertThat(beliefs).hasSize(1);
        assertThat(beliefs.get(0).key()).isEqualTo("poison-location");
        assertThat(beliefs.get(0).currentValue()).isEqualTo("Library");
        assertThat(beliefs.get(0).revisedAtTick()).isEqualTo(-1);
    }

    @Test
    void reviseTracksPriorValue() {
        var store = new BeliefStore();
        store.init("hooded-claw", List.of(
            new SocialConfig.BeliefEntry("poison-location", "Library")
        ));
        store.revise("hooded-claw", "poison-location", "Kitchen", 12);
        var beliefs = store.beliefs("hooded-claw");
        assertThat(beliefs.get(0).currentValue()).isEqualTo("Kitchen");
        assertThat(beliefs.get(0).priorValue()).isEqualTo("Library");
        assertThat(beliefs.get(0).revisedAtTick()).isEqualTo(12);
    }

    @Test
    void unknownAgentReturnsEmpty() {
        var store = new BeliefStore();
        assertThat(store.beliefs("nobody")).isEmpty();
    }

    @Test
    void reviseAddsNewBeliefIfKeyNotPresent() {
        var store = new BeliefStore();
        store.init("hooded-claw", List.of());
        store.revise("hooded-claw", "new-fact", "something", 5);
        var beliefs = store.beliefs("hooded-claw");
        assertThat(beliefs).hasSize(1);
        assertThat(beliefs.get(0).currentValue()).isEqualTo("something");
        assertThat(beliefs.get(0).priorValue()).isNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=BeliefStoreTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure.

- [ ] **Step 3: Implement BeliefStore**

```java
package io.casehub.examples.manor.agent;

import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public final class BeliefStore {

    public record StoredBelief(String key, String currentValue, String priorValue, int revisedAtTick) {}

    private final Map<String, Map<String, StoredBelief>> store = new ConcurrentHashMap<>();

    public void init(String agentId, List<SocialConfig.BeliefEntry> initialBeliefs) {
        var beliefs = new ConcurrentHashMap<String, StoredBelief>();
        for (var b : initialBeliefs) {
            beliefs.put(b.key(), new StoredBelief(b.key(), b.value(), null, -1));
        }
        store.put(agentId, beliefs);
    }

    public List<StoredBelief> beliefs(String agentId) {
        var beliefs = store.get(agentId);
        if (beliefs == null) return List.of();
        return new ArrayList<>(beliefs.values());
    }

    public void revise(String agentId, String key, String newValue, int tick) {
        var beliefs = store.computeIfAbsent(agentId, k -> new ConcurrentHashMap<>());
        var existing = beliefs.get(key);
        String prior = existing != null ? existing.currentValue() : null;
        beliefs.put(key, new StoredBelief(key, newValue, prior, tick));
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=BeliefStoreTest`
Expected: all 4 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/BeliefStore.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/BeliefStoreTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): BeliefStore — per-character beliefs with revision tracking"
```

---

### Task 3: ManorRelationshipTrust — per-relationship trust with decay

Replaces the simple global `ManorTrustProvider` with per-relationship trust scoring.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/ManorRelationshipTrust.java`
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorTrustProvider.java` (use `ide_refactor_safe_delete`)
- Test: `src/test/java/io/casehub/examples/manor/agent/ManorRelationshipTrustTest.java`

**Interfaces:**
- Consumes: action type + target agent from ScenarioOrchestrator tick loop
- Produces: `record(String actorId, String targetId, TrustEvent event)`, `trustLevel(String actorId, String targetId)` returning HIGH/MODERATE/LOW/UNKNOWN, `trustSummaries(String agentId, Collection<String> otherAgentIds)` returning `List<TrustSummary>` for prompt rendering

- [ ] **Step 1: Write failing test**

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class ManorRelationshipTrustTest {

    @Test
    void unknownRelationshipIsUnknown() {
        var trust = new ManorRelationshipTrust();
        assertThat(trust.trustLevel("a", "b")).isEqualTo(ManorRelationshipTrust.TrustLevel.UNKNOWN);
    }

    @Test
    void positiveEventIncreasesTowardHigh() {
        var trust = new ManorRelationshipTrust();
        trust.record("a", "b", ManorRelationshipTrust.TrustEvent.GIVE);
        trust.record("a", "b", ManorRelationshipTrust.TrustEvent.HELP);
        trust.record("a", "b", ManorRelationshipTrust.TrustEvent.HELP);
        assertThat(trust.trustLevel("a", "b")).isEqualTo(ManorRelationshipTrust.TrustLevel.HIGH);
    }

    @Test
    void negativeEventDecreasesTowardLow() {
        var trust = new ManorRelationshipTrust();
        trust.record("a", "b", ManorRelationshipTrust.TrustEvent.STEAL);
        assertThat(trust.trustLevel("a", "b")).isEqualTo(ManorRelationshipTrust.TrustLevel.LOW);
    }

    @Test
    void trustDecaysTowardNeutral() {
        var trust = new ManorRelationshipTrust();
        trust.record("a", "b", ManorRelationshipTrust.TrustEvent.STEAL);
        assertThat(trust.trustLevel("a", "b")).isEqualTo(ManorRelationshipTrust.TrustLevel.LOW);
        trust.applyDecay(10);
        assertThat(trust.trustLevel("a", "b")).isEqualTo(ManorRelationshipTrust.TrustLevel.MODERATE);
    }

    @Test
    void trustSummariesForPrompt() {
        var trust = new ManorRelationshipTrust();
        trust.record("a", "b", ManorRelationshipTrust.TrustEvent.GIVE);
        var summaries = trust.trustSummaries("a", List.of("b", "c"));
        assertThat(summaries).hasSize(2);
        assertThat(summaries.stream().filter(s -> s.targetId().equals("b")).findFirst().get().level())
            .isNotEqualTo(ManorRelationshipTrust.TrustLevel.UNKNOWN);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorRelationshipTrustTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure.

- [ ] **Step 3: Implement ManorRelationshipTrust**

```java
package io.casehub.examples.manor.agent;

import java.util.Collection;
import java.util.List;
import java.util.Map;
import java.util.concurrent.ConcurrentHashMap;

public final class ManorRelationshipTrust {

    public enum TrustLevel { HIGH, MODERATE, LOW, UNKNOWN }
    public enum TrustEvent { GIVE, HELP, STEAL, BETRAY, THREATEN }
    public record TrustSummary(String targetId, TrustLevel level, double score) {}

    private static final double NEUTRAL = 0.5;
    private static final double DECAY_RATE = 0.05;
    private static final Map<TrustEvent, Double> WEIGHTS = Map.of(
        TrustEvent.GIVE, 0.15,
        TrustEvent.HELP, 0.1,
        TrustEvent.STEAL, -0.4,
        TrustEvent.BETRAY, -0.6,
        TrustEvent.THREATEN, -0.2
    );

    private final Map<String, Map<String, Double>> scores = new ConcurrentHashMap<>();

    public void record(String actorId, String targetId, TrustEvent event) {
        scores.computeIfAbsent(actorId, k -> new ConcurrentHashMap<>())
              .merge(targetId, NEUTRAL + WEIGHTS.get(event), (old, delta) ->
                  Math.max(0.0, Math.min(1.0, old + WEIGHTS.get(event))));
    }

    public TrustLevel trustLevel(String actorId, String targetId) {
        var perActor = scores.get(actorId);
        if (perActor == null) return TrustLevel.UNKNOWN;
        Double score = perActor.get(targetId);
        if (score == null) return TrustLevel.UNKNOWN;
        if (score >= 0.7) return TrustLevel.HIGH;
        if (score >= 0.4) return TrustLevel.MODERATE;
        return TrustLevel.LOW;
    }

    public List<TrustSummary> trustSummaries(String agentId, Collection<String> otherAgentIds) {
        return otherAgentIds.stream()
            .map(other -> {
                var level = trustLevel(agentId, other);
                var perActor = scores.get(agentId);
                double score = perActor != null && perActor.get(other) != null ? perActor.get(other) : NEUTRAL;
                return new TrustSummary(other, level, score);
            })
            .toList();
    }

    public void applyDecay(int ticks) {
        scores.values().forEach(perActor ->
            perActor.replaceAll((target, score) -> {
                if (score > NEUTRAL) return Math.max(NEUTRAL, score - DECAY_RATE * ticks);
                if (score < NEUTRAL) return Math.min(NEUTRAL, score + DECAY_RATE * ticks);
                return score;
            })
        );
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=ManorRelationshipTrustTest`
Expected: all 5 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorRelationshipTrust.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorRelationshipTrustTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): ManorRelationshipTrust — per-relationship trust with decay"
```

---

## Batch 2: Rendering + subsystem replacements

### Task 4: SocialCognitionRenderer + ObservationBuilder integration

Renders beliefs, drives, norms, and trust into observation sections. Wires into ObservationBuilder.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/SocialCognitionRenderer.java`
- Modify: `src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java` — add social sections parameter
- Test: `src/test/java/io/casehub/examples/manor/agent/SocialCognitionRendererTest.java`

**Interfaces:**
- Consumes: `SocialConfig` from Task 1, `BeliefStore.beliefs()` from Task 2, `ManorRelationshipTrust.trustSummaries()` from Task 3
- Produces: `List<ObservationSection> renderSocialSections(String agentId, SocialConfig config, BeliefStore beliefStore, ManorRelationshipTrust trust, Collection<String> nearbyAgentIds, Map<String, String> agentNames)`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class SocialCognitionRendererTest {

    @Test
    void rendersDrivesSection() {
        var config = new SocialConfig(
            List.of(new SocialConfig.DriveEntry("scheming", 0.9, "Compelled to scheme")),
            List.of(),
            List.of()
        );
        var beliefStore = new BeliefStore();
        var trust = new ManorRelationshipTrust();
        var sections = SocialCognitionRenderer.renderSocialSections(
            "hc", config, beliefStore, trust, List.of(), Map.of());
        var drivesSection = sections.stream()
            .filter(s -> s.heading().equals("Your Drives"))
            .findFirst();
        assertThat(drivesSection).isPresent();
    }

    @Test
    void rendersBeliefsSectionWithRevision() {
        var config = new SocialConfig(List.of(), List.of(), List.of());
        var beliefStore = new BeliefStore();
        beliefStore.init("hc", List.of(
            new SocialConfig.BeliefEntry("poison", "Library")));
        beliefStore.revise("hc", "poison", "Kitchen", 12);
        var trust = new ManorRelationshipTrust();
        var sections = SocialCognitionRenderer.renderSocialSections(
            "hc", config, beliefStore, trust, List.of(), Map.of());
        var beliefsSection = sections.stream()
            .filter(s -> s.heading().equals("Your Beliefs"))
            .findFirst();
        assertThat(beliefsSection).isPresent();
    }

    @Test
    void rendersNormsSection() {
        var config = new SocialConfig(
            List.of(),
            List.of(new SocialConfig.NormEntry("Never help Penelope", 10)),
            List.of()
        );
        var beliefStore = new BeliefStore();
        var trust = new ManorRelationshipTrust();
        var sections = SocialCognitionRenderer.renderSocialSections(
            "hc", config, beliefStore, trust, List.of(), Map.of());
        var normsSection = sections.stream()
            .filter(s -> s.heading().equals("Your Norms"))
            .findFirst();
        assertThat(normsSection).isPresent();
    }

    @Test
    void rendersTrustSection() {
        var config = new SocialConfig(List.of(), List.of(), List.of());
        var beliefStore = new BeliefStore();
        var trust = new ManorRelationshipTrust();
        trust.record("hc", "penelope", ManorRelationshipTrust.TrustEvent.STEAL);
        var sections = SocialCognitionRenderer.renderSocialSections(
            "hc", config, beliefStore, trust, List.of("penelope"),
            Map.of("penelope", "Penelope Pitstop"));
        var trustSection = sections.stream()
            .filter(s -> s.heading().equals("Your Trust"))
            .findFirst();
        assertThat(trustSection).isPresent();
    }

    @Test
    void omitsSectionsWhenEmpty() {
        var sections = SocialCognitionRenderer.renderSocialSections(
            "hc", SocialConfig.EMPTY, new BeliefStore(), new ManorRelationshipTrust(),
            List.of(), Map.of());
        assertThat(sections).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=SocialCognitionRendererTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure.

- [ ] **Step 3: Implement SocialCognitionRenderer**

```java
package io.casehub.examples.manor.agent;

import io.casehub.blocks.summarisation.observation.affordance.ObservationSection;
import java.util.ArrayList;
import java.util.Collection;
import java.util.List;
import java.util.Map;

public final class SocialCognitionRenderer {

    public static List<ObservationSection> renderSocialSections(
            String agentId,
            SocialConfig config,
            BeliefStore beliefStore,
            ManorRelationshipTrust trust,
            Collection<String> nearbyAgentIds,
            Map<String, String> agentNames) {

        var sections = new ArrayList<ObservationSection>();

        if (!config.drives().isEmpty()) {
            var items = config.drives().stream()
                .map(d -> intensityLabel(d.intensity()).toUpperCase() + " " + d.type() + ": " + d.description())
                .toList();
            sections.add(ObservationSection.items("Your Drives", null, items));
        }

        var beliefs = beliefStore.beliefs(agentId);
        if (!beliefs.isEmpty()) {
            var items = beliefs.stream()
                .map(b -> {
                    if (b.revisedAtTick() >= 0 && b.priorValue() != null) {
                        return "[REVISED tick " + b.revisedAtTick() + "] You now believe: "
                            + b.key() + " — " + b.currentValue() + " (was: " + b.priorValue() + ")";
                    }
                    return "You believe: " + b.key() + " — " + b.currentValue();
                })
                .toList();
            sections.add(ObservationSection.items("Your Beliefs", null, items));
        }

        if (!config.norms().isEmpty()) {
            var sorted = config.norms().stream()
                .sorted((a, b) -> Integer.compare(b.priority(), a.priority()))
                .toList();
            var items = sorted.stream()
                .map(n -> n.rule() + " (priority: " + priorityLabel(n.priority()) + ")")
                .toList();
            sections.add(ObservationSection.items("Your Norms", null, items));
        }

        if (!nearbyAgentIds.isEmpty()) {
            var summaries = trust.trustSummaries(agentId, nearbyAgentIds);
            var relevant = summaries.stream()
                .filter(s -> s.level() != ManorRelationshipTrust.TrustLevel.UNKNOWN)
                .toList();
            if (!relevant.isEmpty()) {
                var items = relevant.stream()
                    .map(s -> agentNames.getOrDefault(s.targetId(), s.targetId())
                        + ": " + s.level().name())
                    .toList();
                sections.add(ObservationSection.items("Your Trust", null, items));
            }
        }

        return sections;
    }

    private static String intensityLabel(double intensity) {
        if (intensity >= 0.8) return "strong";
        if (intensity >= 0.5) return "moderate";
        return "mild";
    }

    private static String priorityLabel(int priority) {
        if (priority >= 8) return "high";
        if (priority >= 4) return "medium";
        return "low";
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=SocialCognitionRendererTest`
Expected: all 5 tests PASS.

- [ ] **Step 5: Update ObservationBuilder to accept social sections**

Add a `socialSections` parameter to `buildObservation()`. Insert social sections after inventory, before goals:

```java
// In ObservationBuilder.buildObservation(), add parameter:
//   List<ObservationSection> socialSections
// After line: sections.add(inventorySection(character));
// Add: sections.addAll(socialSections);
```

Use `ide_replace_member` to update the method signature and body.

- [ ] **Step 6: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all existing tests PASS (callers of buildObservation pass `List.of()` for now).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialCognitionRenderer.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/SocialCognitionRendererTest.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/ObservationBuilder.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): SocialCognitionRenderer + ObservationBuilder integration"
```

---

### Task 5: BlocksDispositionAdapter — replace ManorDispositionRecorder with blocks memory scoring

Delegates importance scoring to blocks SurpriseScorer and ArousalScorer. Replaces ManorDispositionRecorder + ManorPersonalityEvolution.

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/agent/BlocksDispositionAdapter.java`
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorDispositionRecorder.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorPersonalityEvolution.java` (use `ide_refactor_safe_delete`)
- Test: `src/test/java/io/casehub/examples/manor/agent/BlocksDispositionAdapterTest.java`

**Interfaces:**
- Consumes: `blocks.memory.SurpriseScorer`, `blocks.memory.ArousalScorer`, `blocks.memory.ConfidenceScorer`
- Produces: `double computeImportance(AgentResponse response, String agentId)`, `void record(String agentId, ActionType action, ActionResult result)`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.model.ActionType;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class BlocksDispositionAdapterTest {

    @Test
    void stealHasHighImportance() {
        var adapter = new BlocksDispositionAdapter();
        double importance = adapter.computeImportance(ActionType.STEAL);
        assertThat(importance).isGreaterThanOrEqualTo(0.8);
    }

    @Test
    void waitHasLowImportance() {
        var adapter = new BlocksDispositionAdapter();
        double importance = adapter.computeImportance(ActionType.WAIT);
        assertThat(importance).isLessThan(0.3);
    }

    @Test
    void lookHasMinimalImportance() {
        var adapter = new BlocksDispositionAdapter();
        double importance = adapter.computeImportance(ActionType.LOOK);
        assertThat(importance).isLessThan(0.4);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=BlocksDispositionAdapterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure.

- [ ] **Step 3: Implement BlocksDispositionAdapter**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.model.ActionType;

public final class BlocksDispositionAdapter {

    public double computeImportance(ActionType actionType) {
        if (actionType == null) return 0.5;
        return switch (actionType) {
            case STEAL -> 0.9;
            case USE -> 0.8;
            case TAKE, GIVE, PULL_ASIDE -> 0.7;
            case INTERACT -> 0.6;
            case MOVE -> 0.3;
            case LOOK -> 0.2;
            case WAIT -> 0.1;
        };
    }
}
```

Note: initial implementation mirrors the existing `importanceForAction()` switch. Blocks `SurpriseScorer`/`ArousalScorer` integration happens in the ScenarioOrchestrator wiring task where we have access to the character's history needed for context-aware scoring.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=BlocksDispositionAdapterTest`
Expected: all 3 tests PASS.

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/agent/BlocksDispositionAdapter.java wacky-manor/src/test/java/io/casehub/examples/manor/agent/BlocksDispositionAdapterTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): BlocksDispositionAdapter — importance scoring for disposition replacement"
```

---

## Batch 3: Orchestrator wiring + character profiles

### Task 6: Wire social cognition into ScenarioOrchestrator + character descriptors + belief revision

The integration task. Wires all new services into the tick loop, updates character YAML descriptors with social config, adds BeliefRevisionService, removes old trust/disposition types.

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` — wire new services, replace trust/disposition
- Create: `src/main/java/io/casehub/examples/manor/agent/BeliefRevisionService.java`
- Modify: `src/main/resources/META-INF/eidos/descriptors-composite.yaml` — add social sections to 5 characters
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorTrustProvider.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorDispositionRecorder.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/casehub/examples/manor/agent/ManorPersonalityEvolution.java` (use `ide_refactor_safe_delete`)
- Test: `src/test/java/io/casehub/examples/manor/agent/BeliefRevisionServiceTest.java`

**Interfaces:**
- Consumes: All from Tasks 1–5
- Produces: Fully wired social cognition in the tick loop

- [ ] **Step 1: Write BeliefRevisionService failing test**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.model.ActionResult;
import io.casehub.examples.manor.model.ActionType;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.assertThat;

class BeliefRevisionServiceTest {

    @Test
    void revisesLocationBeliefOnMoveFindingItem() {
        var beliefStore = new BeliefStore();
        beliefStore.init("hc", List.of(
            new SocialConfig.BeliefEntry("poison-location", "Library")
        ));
        var service = new BeliefRevisionService(beliefStore);
        service.reviseOnActionResult("hc", ActionType.LOOK, "poison",
            new ActionResult.Failed("Not found here"), "Kitchen", 5);
        var beliefs = beliefStore.beliefs("hc");
        assertThat(beliefs.get(0).currentValue()).isEqualTo("Library");
    }

    @Test
    void revisesBeliefWhenItemFoundInDifferentRoom() {
        var beliefStore = new BeliefStore();
        beliefStore.init("hc", List.of(
            new SocialConfig.BeliefEntry("poison-location", "Library")
        ));
        var service = new BeliefRevisionService(beliefStore);
        service.reviseOnObservation("hc", "poison", "Kitchen", 8);
        var beliefs = beliefStore.beliefs("hc");
        assertThat(beliefs.get(0).currentValue()).isEqualTo("Kitchen");
        assertThat(beliefs.get(0).priorValue()).isEqualTo("Library");
        assertThat(beliefs.get(0).revisedAtTick()).isEqualTo(8);
    }

    @Test
    void noRevisionWhenBeliefMatches() {
        var beliefStore = new BeliefStore();
        beliefStore.init("hc", List.of(
            new SocialConfig.BeliefEntry("poison-location", "Library")
        ));
        var service = new BeliefRevisionService(beliefStore);
        service.reviseOnObservation("hc", "poison", "Library", 3);
        var beliefs = beliefStore.beliefs("hc");
        assertThat(beliefs.get(0).revisedAtTick()).isEqualTo(-1);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=BeliefRevisionServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure.

- [ ] **Step 3: Implement BeliefRevisionService**

```java
package io.casehub.examples.manor.agent;

import io.casehub.examples.manor.model.ActionResult;
import io.casehub.examples.manor.model.ActionType;

public final class BeliefRevisionService {

    private final BeliefStore beliefStore;

    public BeliefRevisionService(BeliefStore beliefStore) {
        this.beliefStore = beliefStore;
    }

    public void reviseOnObservation(String agentId, String itemKey, String observedRoom, int tick) {
        var beliefs = beliefStore.beliefs(agentId);
        for (var belief : beliefs) {
            if (belief.key().contains(itemKey) && !belief.currentValue().equals(observedRoom)) {
                beliefStore.revise(agentId, belief.key(), observedRoom, tick);
            }
        }
    }

    public void reviseOnActionResult(String agentId, ActionType action, String target,
                                      ActionResult result, String room, int tick) {
        if (result instanceof ActionResult.Failed) {
            var beliefs = beliefStore.beliefs(agentId);
            for (var belief : beliefs) {
                if (belief.key().contains(target) && belief.currentValue().equals(room)) {
                    beliefStore.revise(agentId, belief.key(),
                        "not in " + room + " (discovered tick " + tick + ")", tick);
                }
            }
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml -Dtest=BeliefRevisionServiceTest`
Expected: all 3 tests PASS.

- [ ] **Step 5: Add social sections to 5 character Eidos descriptors**

Update `src/main/resources/META-INF/eidos/descriptors-composite.yaml`. Add `extensionData.social` to each of the 5 core characters:

**hooded-claw:** drives: scheming (0.9), self-preservation (0.7), dominance (0.6). Norms: never help Penelope (10), maintain charm in public (5), protect schemes (8). Beliefs: penelope-awareness, peter-threat, poison-location.

**penelope-pitstop:** drives: cooperation (0.8), curiosity (0.7), self-preservation (0.5). Norms: help anyone in need (10), avoid violence (8), trust until proven wrong (6). Beliefs: everyone-good, manor-mystery.

**peter-perfect:** drives: loyalty (0.9), cooperation (0.6), self-preservation (0.4). Norms: protect Penelope (10), confront threats (7), never steal (9). Beliefs: hooded-claw-suspicious, penelope-needs-protection.

**muttley:** drives: loyalty (0.7), self-preservation (0.8), curiosity (0.5). Norms: follow Dastardly's lead (8), avoid confrontation (6), hoard interesting items (4). Beliefs: dastardly-has-plan, others-unpredictable.

**dick-dastardly:** drives: scheming (0.7), dominance (0.8), self-preservation (0.6). Norms: maintain authority over Muttley (9), outdo Hooded Claw (7), never appear weak (8). Beliefs: hooded-claw-rival, penelope-obstacle.

Format in YAML under `extensionData:` key for each descriptor.

- [ ] **Step 6: Wire into ScenarioOrchestrator**

Use `ide_replace_member` on `ScenarioOrchestrator.runScenario()` and `runAutonomousTicks()`:

1. Instantiate `BeliefStore`, `BeliefRevisionService`, `SocialCognitionLoader`, `ManorRelationshipTrust`
2. Init beliefs for each active character from their Eidos extensionData
3. In the per-character tick: load social config, render social sections, pass to ObservationBuilder
4. After action resolution: record trust events on ManorRelationshipTrust (replacing ManorTrustProvider calls)
5. After action resolution: call BeliefRevisionService
6. Replace `importanceForAction()` with `BlocksDispositionAdapter.computeImportance()`
7. Remove ManorTrustProvider, ManorDispositionRecorder, ManorPersonalityEvolution instantiation

- [ ] **Step 7: Delete old types**

Use `ide_refactor_safe_delete` for:
- `ManorTrustProvider`
- `ManorDispositionRecorder`
- `ManorPersonalityEvolution`

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests PASS.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#52): wire social cognition into ScenarioOrchestrator + character descriptors"
```

---

## Deferred

- **LLM eval tests:** Add after social cognition is wired and verified manually. These are non-deterministic and should be added once the deterministic tests confirm correctness.
- **Blocks SurpriseScorer/ArousalScorer integration:** The initial BlocksDispositionAdapter mirrors the existing importance switch. Context-aware scoring using blocks memory scorers requires per-character history, which is better added incrementally after the foundation works.
- **Norm context filtering:** Initial implementation renders all norms. Context-based filtering (only show norms relevant to current situation) is an enhancement after the basic norms are validated.

## References

- [2026-09-11-social-cognition-design.md] — design spec this plan implements
- [ScenarioOrchestrator.java:241-434] — autonomous tick loop being modified
- [ObservationBuilder.java:1-88] — observation rendering being extended
- [ManorTrustProvider.java] — trust subsystem being replaced
- [ManorDispositionRecorder.java] — disposition subsystem being replaced
- [descriptors-composite.yaml] — Eidos character descriptors
- [blocks/blocks/.../memory/SurpriseScorer.java] — blocks memory scoring
- [blocks/blocks/.../agentic/belief/] — blocks belief model
- [blocks/blocks/.../normative/] — blocks normative reasoning
- [blocks/blocks/.../agentic/social/prompt/] — DrivePromptSection, SocialPromptAssembler
- [GE-20260816-6635e1] — ChannelObserver bridging pattern
- [GitHub #52] — focal issue
