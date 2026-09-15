# SocialConfig to YAML Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #56 — Move SocialConfig from hardcoded Java to Eidos extensionData
**Issue group:** #54, #55, #56, #57, #59, #60

**Goal:** Move hardcoded SocialConfig data (drives, norms, beliefs for 5 characters) from a Java map to a YAML file parsed at startup.

**Architecture:** A standalone `social-config.yaml` in `META-INF/eidos/` holds all character social config. A static `ManorSocialConfigLoader.load()` reads it from the classpath via Jackson YAML. ScenarioOrchestrator calls the loader once at startup, replacing `SocialConfig.forCharacter()`. The hardcoded `CONFIGS` map is removed from SocialConfig.

**Tech Stack:** Java 26, Jackson YAML (already on classpath via Quarkus), SnakeYAML engine

## Global Constraints

- YAML file location: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`
- SocialConfig sub-records (Drive, NormEntry, InitialBelief) are unchanged — only the data source changes
- Jackson `ObjectMapper` with `YAMLFactory` for parsing (Quarkus provides this)
- `initial-beliefs` in YAML maps to `initialBeliefs` in Java — use Jackson's kebab-case naming strategy or explicit `@JsonProperty`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`

---

## Batch 1: YAML file + loader + wiring

### Task 1: ManorSocialConfigLoader + social-config.yaml

**Files:**
- Create: `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java`

**Interfaces:**
- Consumes: `SocialConfig(List<Drive>, List<NormEntry>, List<InitialBelief>)` — existing record
- Produces: `ManorSocialConfigLoader.load()` → `Map<String, SocialConfig>` (default resource path)
- Produces: `ManorSocialConfigLoader.load(String resourcePath)` → `Map<String, SocialConfig>` (testable overload)

- [ ] **Step 1: Create social-config.yaml**

Create `wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml` with all 5 characters' data from the current hardcoded `CONFIGS` map in SocialConfig.java:

```yaml
hooded-claw:
  drives:
    - type: scheming
      intensity: 0.9
      description: "Compelled to hatch elaborate plans against Penelope"
    - type: self-preservation
      intensity: 0.7
      description: "Avoids direct confrontation, prefers subterfuge"
    - type: dominance
      intensity: 0.6
      description: "Must be the most powerful person in every room"
  norms:
    - rule: "Never help Penelope directly"
      priority: 10
    - rule: "Maintain a veneer of charm in public"
      priority: 5
    - rule: "Protect personal schemes from discovery"
      priority: 8
  initial-beliefs:
    - key: penelope-awareness
      value: "Penelope is naive and trusts too easily"
    - key: peter-threat
      value: "Peter Perfect is protective but predictable"

penelope-pitstop:
  drives:
    - type: curiosity
      intensity: 0.7
      description: "Drawn to puzzles and mysteries"
    - type: social-harmony
      intensity: 0.8
      description: "Wants everyone to get along"
    - type: adventure
      intensity: 0.6
      description: "Delights in new experiences"
  norms:
    - rule: "Trust everyone until proven otherwise"
      priority: 8
    - rule: "Help those in need"
      priority: 9
    - rule: "Stay positive and encouraging"
      priority: 5
  initial-beliefs:
    - key: sneekly-trust
      value: "Sylvester Sneekly is a helpful estate manager"
    - key: general-trust
      value: "Everyone here means well"

peter-perfect:
  drives:
    - type: gallantry
      intensity: 0.9
      description: "Must protect and impress Penelope"
    - type: proving-worth
      intensity: 0.7
      description: "Needs to demonstrate heroic competence"
    - type: protection
      intensity: 0.8
      description: "Driven to shield others from danger"
  norms:
    - rule: "Always prepare before acting"
      priority: 8
    - rule: "Protect Penelope at all costs"
      priority: 9
    - rule: "Give people the benefit of the doubt"
      priority: 5
  initial-beliefs:
    - key: penelope-needs
      value: "Penelope needs protection"
    - key: planning-value
      value: "Planning beats improvisation"

dick-dastardly:
  drives:
    - type: greed
      intensity: 0.9
      description: "Wants the treasure more than anything"
    - type: recognition
      intensity: 0.8
      description: "Craves medals and acknowledgment"
    - type: scheming
      intensity: 0.7
      description: "Enjoys outwitting others"
  norms:
    - rule: "Lie about everything"
      priority: 10
    - rule: "Never do honest work"
      priority: 8
    - rule: "Take credit for others' achievements"
      priority: 7
  initial-beliefs:
    - key: superiority
      value: "Everyone else is a fool"
    - key: method
      value: "Cheating is smarter than working"

ant-hill-mob:
  drives:
    - type: loyalty
      intensity: 0.9
      description: "Fiercely devoted to protecting Penelope"
    - type: protection
      intensity: 0.8
      description: "Must keep Penelope safe from harm"
    - type: suspicion
      intensity: 0.6
      description: "Something about Sneekly ain't right"
  norms:
    - rule: "Protect Penelope always"
      priority: 10
    - rule: "Trust your gut feeling"
      priority: 7
    - rule: "Never accuse anyone directly"
      priority: 5
  initial-beliefs:
    - key: sneekly-suspicion
      value: "Something about Sneekly ain't right"
    - key: penelope-safety
      value: "Penelope needs protecting from herself"
```

- [ ] **Step 2: Write failing tests for ManorSocialConfigLoader**

Create `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java`:

```java
package io.casehub.examples.manor.agent;

import org.junit.jupiter.api.Test;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class ManorSocialConfigLoaderTest {

    @Test
    void loadsAllFiveCharacters() {
        var configs = ManorSocialConfigLoader.load();
        assertThat(configs).hasSize(5);
        assertThat(configs).containsKeys(
                "hooded-claw", "penelope-pitstop", "peter-perfect",
                "dick-dastardly", "ant-hill-mob");
    }

    @Test
    void hoodedClaw_drivesMatchHardcoded() {
        var hc = ManorSocialConfigLoader.load().get("hooded-claw");
        assertThat(hc.drives()).hasSize(3);
        var scheming = hc.drives().stream()
                .filter(d -> d.type().equals("scheming")).findFirst().orElseThrow();
        assertThat(scheming.intensity()).isEqualTo(0.9);
        assertThat(scheming.description()).contains("elaborate plans");
    }

    @Test
    void hoodedClaw_normsMatchHardcoded() {
        var hc = ManorSocialConfigLoader.load().get("hooded-claw");
        assertThat(hc.norms()).hasSize(3);
        var topNorm = hc.norms().stream()
                .filter(n -> n.priority() == 10).findFirst().orElseThrow();
        assertThat(topNorm.rule()).isEqualTo("Never help Penelope directly");
    }

    @Test
    void hoodedClaw_initialBeliefsMatchHardcoded() {
        var hc = ManorSocialConfigLoader.load().get("hooded-claw");
        assertThat(hc.initialBeliefs()).hasSize(2);
        var penBelief = hc.initialBeliefs().stream()
                .filter(b -> b.key().equals("penelope-awareness")).findFirst().orElseThrow();
        assertThat(penBelief.value()).isEqualTo("Penelope is naive and trusts too easily");
    }

    @Test
    void penelopePitstop_loaded() {
        var pp = ManorSocialConfigLoader.load().get("penelope-pitstop");
        assertThat(pp.drives()).hasSize(3);
        assertThat(pp.norms()).hasSize(3);
        assertThat(pp.initialBeliefs()).hasSize(2);
    }

    @Test
    void unknownCharacter_notInMap() {
        var configs = ManorSocialConfigLoader.load();
        assertThat(configs).doesNotContainKey("muttley");
    }

    @Test
    void missingResource_throwsIllegalState() {
        assertThatThrownBy(() -> ManorSocialConfigLoader.load("nonexistent.yaml"))
                .isInstanceOf(IllegalStateException.class);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorSocialConfigLoaderTest`
Expected: FAIL — `ManorSocialConfigLoader` does not exist

- [ ] **Step 4: Implement ManorSocialConfigLoader**

Create `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java` via `ide_create_file` (or Write as fallback for new files):

```java
package io.casehub.examples.manor.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.databind.PropertyNamingStrategies;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;

import java.io.IOException;
import java.io.InputStream;
import java.util.Collections;
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public final class ManorSocialConfigLoader {

    private static final String DEFAULT_RESOURCE = "META-INF/eidos/social-config.yaml";

    private ManorSocialConfigLoader() {}

    public static Map<String, SocialConfig> load() {
        return load(DEFAULT_RESOURCE);
    }

    public static Map<String, SocialConfig> load(String resourcePath) {
        InputStream is = Thread.currentThread().getContextClassLoader()
                .getResourceAsStream(resourcePath);
        if (is == null) {
            throw new IllegalStateException("Social config not found: " + resourcePath);
        }

        var mapper = new ObjectMapper(new YAMLFactory())
                .setPropertyNamingStrategy(PropertyNamingStrategies.KEBAB_CASE);

        try (is) {
            @SuppressWarnings("unchecked")
            Map<String, Map<String, Object>> raw = mapper.readValue(is, Map.class);

            var result = new HashMap<String, SocialConfig>();
            for (var entry : raw.entrySet()) {
                result.put(entry.getKey(), parseCharacterConfig(entry.getValue(), mapper));
            }
            return Collections.unmodifiableMap(result);
        } catch (IOException e) {
            throw new IllegalStateException("Failed to parse social config: " + resourcePath, e);
        }
    }

    @SuppressWarnings("unchecked")
    private static SocialConfig parseCharacterConfig(Map<String, Object> raw, ObjectMapper mapper) {
        var drives = raw.containsKey("drives")
                ? ((List<Map<String, Object>>) raw.get("drives")).stream()
                    .map(m -> new SocialConfig.Drive(
                            (String) m.get("type"),
                            ((Number) m.get("intensity")).doubleValue(),
                            (String) m.get("description")))
                    .toList()
                : List.<SocialConfig.Drive>of();

        var norms = raw.containsKey("norms")
                ? ((List<Map<String, Object>>) raw.get("norms")).stream()
                    .map(m -> new SocialConfig.NormEntry(
                            (String) m.get("rule"),
                            ((Number) m.get("priority")).intValue()))
                    .toList()
                : List.<SocialConfig.NormEntry>of();

        var beliefs = raw.containsKey("initial-beliefs")
                ? ((List<Map<String, Object>>) raw.get("initial-beliefs")).stream()
                    .map(m -> new SocialConfig.InitialBelief(
                            (String) m.get("key"),
                            (String) m.get("value")))
                    .toList()
                : List.<SocialConfig.InitialBelief>of();

        return new SocialConfig(drives, norms, beliefs);
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorSocialConfigLoaderTest`
Expected: PASS — all 7 tests

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/resources/META-INF/eidos/social-config.yaml
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorSocialConfigLoader.java
git add wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorSocialConfigLoaderTest.java
git commit -m "feat(#56): ManorSocialConfigLoader + social-config.yaml — YAML-based character config

Refs #56

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 2: Wire loader into ScenarioOrchestrator + remove hardcoded map

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java`
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java`

**Interfaces:**
- Consumes: `ManorSocialConfigLoader.load()` → `Map<String, SocialConfig>` from Task 1
- Consumes: `SocialConfig.empty()` — retained for characters not in the YAML

- [ ] **Step 1: Update ScenarioOrchestrator to use loader**

In `ScenarioOrchestrator.runScenario()`, find the character initialization loop. Add the loader call before the loop and replace the `SocialConfig.forCharacter()` call.

Find (around line 140-160 in runScenario):
```java
var socialCfg = SocialConfig.forCharacter(entry.getKey());
```

Replace with:
```java
var socialCfg = socialConfigs.getOrDefault(entry.getKey(), SocialConfig.empty());
```

Add the loader call before the character loop (before the `for (var entry : world.characters().entrySet())` line):
```java
var socialConfigs = ManorSocialConfigLoader.load();
```

Use `ide_replace_text_in_file` for the forCharacter replacement, and `ide_replace_text_in_file` to add the loader call before the loop.

- [ ] **Step 2: Remove hardcoded CONFIGS from SocialConfig**

Remove from `SocialConfig.java`:
- The `CONFIGS` map (lines 44-109)
- The `forCharacter(String agentId)` method (lines 111-113)
- The `hasConfig(String agentId)` method (lines 115-117)
- The `import java.util.Map;` (no longer needed — check if Map is used elsewhere in the record first)

Keep:
- The record definition with Drive, NormEntry, InitialBelief sub-records
- The `empty()` factory method
- The compact constructor with null-safe List.copyOf

Use `ide_replace_member` to remove the static methods and field.

- [ ] **Step 3: Fix any compilation errors from removed methods**

Check if any other code calls `SocialConfig.forCharacter()` or `SocialConfig.hasConfig()`:

Use `ide_find_references` on `forCharacter` and `hasConfig` in SocialConfig. If found in test code, update those tests to use `ManorSocialConfigLoader.load().get(agentId)` or `ManorSocialConfigLoader.load().containsKey(agentId)`.

- [ ] **Step 4: Verify compilation**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn compile -pl wacky-manor`
Expected: BUILD SUCCESS

- [ ] **Step 5: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: All tests pass (same baseline as before — 450 tests, 1 pre-existing error)

- [ ] **Step 6: Commit**

```bash
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java
git add wacky-manor/src/main/java/io/casehub/examples/manor/agent/SocialConfig.java
git commit -m "feat(#56): wire ManorSocialConfigLoader into ScenarioOrchestrator, remove hardcoded CONFIGS

SocialConfig is now a pure data record. Character social config loaded
from META-INF/eidos/social-config.yaml at scenario start.

Closes #56

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-09-15-socialconfig-yaml-design.md] — design spec this plan implements
- [SocialConfig.java:44-113] — hardcoded CONFIGS map to be removed
- [ScenarioOrchestrator.java:158] — forCharacter() call site
- [AgentDescriptor.java] — no extensionData field (drives D1 decision)
- [GitHub #56] — Move SocialConfig from hardcoded Java to Eidos extensionData
- [GitHub #53] — Phase B epic
