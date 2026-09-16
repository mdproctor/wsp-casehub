# Trust Evolution Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #65 — Trust evolution from experience
**Issue group:** #65

**Goal:** Wire the ledger's Bayesian Beta trust model into the consolidation pipeline so characters develop per-relationship trust from gameplay actions.

**Architecture:** Four-module split — blocks-core (config records), blocks-agentic-yaml (YAML spec), blocks (CDI event + recorder), neocortex-mindmap-intelligence (consolidation phase). Wacky-manor provides YAML config + CDI producer + orchestrator wiring. Trust scores written to person-entity overlay nodes during consolidation, read during observation rendering.

**Tech Stack:** Quarkus CDI, ledger TrustScoreComputer (Bayesian Beta), neocortex MindMapStore, Jackson YAML

## Global Constraints

- Java 26 (`JAVA_HOME=$(/usr/libexec/java_home -v 26)`)
- Build: `mvn clean install -pl <module> -s slot-settings.xml` (never without `-pl`)
- Three repos in slot 196: examples, blocks, neocortex
- Pre-release: breaking changes cost nothing
- IntelliJ MCP required for all Java edits
- `PlainLedgerEntry` from `casehub-ledger-runtime` — concrete LedgerEntry subclass (ADR-0016)
- `AttestationVerdict` values: SOUND, ENDORSED (positive), FLAGGED, CHALLENGED (negative)
- All consolidation phases use `@Priority` for ordering — existing: AccessFrequency(10), Experience(15), MergeDetection(20), SchemaDiscovery(25), CommunitySummary(30), CuriosityRefresh(40)

---

## Batch 1: Foundation — config records and constants (blocks-core)

### Task 1: TrustEvolutionConfig + OverlayTrustPropertyModel in blocks-core

**Files:**
- Create: `blocks-core/src/main/java/io/casehub/blocks/trust/TrustEvolutionConfig.java`
- Create: `blocks-core/src/main/java/io/casehub/blocks/trust/OverlayTrustPropertyModel.java`
- Test: `blocks-core/src/test/java/io/casehub/blocks/trust/TrustEvolutionConfigTest.java`

**Interfaces:**
- Consumes: `io.casehub.ledger.api.model.AttestationVerdict` (from ledger-api, already a blocks-core dep)
- Produces: `TrustEvolutionConfig` record (consumed by TrustEventRecorder in Task 3, TrustConsolidationPhase in Task 4, CharacterCognition rendering in Task 6)
- Produces: `OverlayTrustPropertyModel` constants (consumed by TrustConsolidationPhase in Task 4, observation rendering in Task 6)

- [ ] **Step 1: Write failing test for TrustEvolutionConfig**

```java
package io.casehub.blocks.trust;

import io.casehub.ledger.api.model.AttestationVerdict;
import org.junit.jupiter.api.Test;
import java.util.List;
import static org.assertj.core.api.Assertions.*;

class TrustEvolutionConfigTest {

    @Test
    void constructsWithAllFields() {
        var config = new TrustEvolutionConfig(
            List.of(new TrustEvolutionConfig.TrustEventMapping(
                "STEAL", AttestationVerdict.FLAGGED, 0.9, 0.5)),
            new TrustEvolutionConfig.ScoringConfig(30, 1.5),
            new TrustEvolutionConfig.ConsolidationConfig("trust-score", 0.15),
            new TrustEvolutionConfig.LevelConfig(0.7, 0.4, 0.2));

        assertThat(config.events()).hasSize(1);
        assertThat(config.events().get(0).actionType()).isEqualTo("STEAL");
        assertThat(config.events().get(0).verdict()).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(config.scoring().decayHalfLifeDays()).isEqualTo(30);
        assertThat(config.scoring().negativeDecayMultiplier()).isEqualTo(1.5);
        assertThat(config.consolidation().significantChangeThreshold()).isEqualTo(0.15);
        assertThat(config.levels().high()).isEqualTo(0.7);
    }

    @Test
    void findsMappingByActionType() {
        var config = new TrustEvolutionConfig(
            List.of(
                new TrustEvolutionConfig.TrustEventMapping("STEAL", AttestationVerdict.FLAGGED, 0.9, 0.5),
                new TrustEvolutionConfig.TrustEventMapping("GIVE", AttestationVerdict.SOUND, 0.7, 0.3)),
            new TrustEvolutionConfig.ScoringConfig(30, 1.5),
            new TrustEvolutionConfig.ConsolidationConfig("trust-score", 0.15),
            new TrustEvolutionConfig.LevelConfig(0.7, 0.4, 0.2));

        assertThat(config.findMapping("STEAL")).isPresent();
        assertThat(config.findMapping("STEAL").get().verdict()).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(config.findMapping("MOVE")).isEmpty();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=TrustEvolutionConfigTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure — `TrustEvolutionConfig` doesn't exist

- [ ] **Step 3: Create TrustEvolutionConfig record**

Use `ide_create_file` for the new file:

```java
package io.casehub.blocks.trust;

import io.casehub.ledger.api.model.AttestationVerdict;
import java.util.List;
import java.util.Objects;
import java.util.Optional;

public record TrustEvolutionConfig(
    List<TrustEventMapping> events,
    ScoringConfig scoring,
    ConsolidationConfig consolidation,
    LevelConfig levels
) {
    public TrustEvolutionConfig {
        events = List.copyOf(Objects.requireNonNull(events));
        Objects.requireNonNull(scoring);
        Objects.requireNonNull(consolidation);
        Objects.requireNonNull(levels);
    }

    public Optional<TrustEventMapping> findMapping(String actionType) {
        return events.stream()
            .filter(m -> m.actionType().equals(actionType))
            .findFirst();
    }

    public record TrustEventMapping(
        String actionType,
        AttestationVerdict verdict,
        double confidence,
        double witnessConfidence
    ) {
        public TrustEventMapping {
            Objects.requireNonNull(actionType);
            Objects.requireNonNull(verdict);
        }
    }

    public record ScoringConfig(int decayHalfLifeDays, double negativeDecayMultiplier) {
        public ScoringConfig {
            if (decayHalfLifeDays <= 0) throw new IllegalArgumentException("decayHalfLifeDays must be positive");
            if (negativeDecayMultiplier <= 0) throw new IllegalArgumentException("negativeDecayMultiplier must be positive");
        }
    }

    public record ConsolidationConfig(String overlayProperty, double significantChangeThreshold) {
        public ConsolidationConfig {
            Objects.requireNonNull(overlayProperty);
            if (significantChangeThreshold < 0 || significantChangeThreshold > 1)
                throw new IllegalArgumentException("significantChangeThreshold must be in [0,1]");
        }
    }

    public record LevelConfig(double high, double moderate, double low) {
        public LevelConfig {
            if (high < moderate || moderate < low)
                throw new IllegalArgumentException("levels must be high >= moderate >= low");
        }
    }
}
```

- [ ] **Step 4: Create OverlayTrustPropertyModel**

Use `ide_create_file`:

```java
package io.casehub.blocks.trust;

public final class OverlayTrustPropertyModel {
    public static final String TRUST_SCORE = "trust-score";
    public static final String TRUST_ALPHA = "trust-alpha";
    public static final String TRUST_BETA = "trust-beta";
    public static final String TRUST_LAST_RENDERED = "trust-last-rendered";

    private OverlayTrustPropertyModel() {}
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks-core -s slot-settings.xml -Dtest=TrustEvolutionConfigTest`
Expected: PASS

- [ ] **Step 6: Verify with ide_diagnostics**

Run `ide_diagnostics` on blocks-core to check for compilation errors.

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks-core/src/main/java/io/casehub/blocks/trust/ blocks-core/src/test/java/io/casehub/blocks/trust/
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#65): TrustEvolutionConfig + OverlayTrustPropertyModel in blocks-core

Refs casehubio/examples#65

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: CDI event layer (blocks) + YAML spec (blocks-agentic-yaml)

### Task 2: TrustRelevantAction CDI event + TrustEventRecorder in blocks

**Files:**
- Create: `blocks/src/main/java/io/casehub/blocks/trust/TrustRelevantAction.java`
- Create: `blocks/src/main/java/io/casehub/blocks/trust/TrustEventRecorder.java`
- Test: `blocks/src/test/java/io/casehub/blocks/trust/TrustEventRecorderTest.java`

**Interfaces:**
- Consumes: `TrustEvolutionConfig` (from Task 1), `LedgerEntryRepository` (ledger SPI), `PlainLedgerEntry` (ledger runtime), `LedgerAttestation` (ledger api)
- Produces: `TrustRelevantAction` CDI event type (fired by ScenarioOrchestrator in Task 5), `TrustEventRecorder` CDI observer (creates ledger entries)

- [ ] **Step 1: Write failing test for TrustEventRecorder**

```java
package io.casehub.blocks.trust;

import io.casehub.blocks.trust.TrustEvolutionConfig.*;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.ArrayList;
import java.util.List;
import java.util.UUID;

import static org.assertj.core.api.Assertions.*;

class TrustEventRecorderTest {

    private CapturingLedgerRepo repo;
    private TrustEventRecorder recorder;
    private TrustEvolutionConfig config;

    @BeforeEach
    void setUp() {
        repo = new CapturingLedgerRepo();
        config = new TrustEvolutionConfig(
            List.of(
                new TrustEventMapping("STEAL", AttestationVerdict.FLAGGED, 0.9, 0.5),
                new TrustEventMapping("GIVE", AttestationVerdict.SOUND, 0.7, 0.3)),
            new ScoringConfig(30, 1.5),
            new ConsolidationConfig("trust-score", 0.15),
            new LevelConfig(0.7, 0.4, 0.2));
        recorder = new TrustEventRecorder(config, repo);
    }

    @Test
    void createsEntryAndAttestationForMappedAction() {
        var event = new TrustRelevantAction(
            "hooded-claw", "penelope-pitstop", "STEAL", "success",
            List.of("peter-perfect"), "test-tenant");
        recorder.onTrustRelevantAction(event);

        assertThat(repo.savedEntries).hasSize(1);
        LedgerEntry entry = repo.savedEntries.get(0);
        assertThat(entry.actorId).isEqualTo("hooded-claw");
        assertThat(entry.entryType).isEqualTo(LedgerEntryType.EVENT);
        assertThat(entry.subjectId).isEqualTo(
            UUID.nameUUIDFromBytes("penelope-pitstop".getBytes()));

        // target + 1 witness = 2 attestations
        assertThat(repo.savedAttestations).hasSize(2);

        var targetAttestation = repo.savedAttestations.stream()
            .filter(a -> "penelope-pitstop".equals(a.attestorId)).findFirst().orElseThrow();
        assertThat(targetAttestation.verdict).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(targetAttestation.confidence).isEqualTo(0.9);

        var witnessAttestation = repo.savedAttestations.stream()
            .filter(a -> "peter-perfect".equals(a.attestorId)).findFirst().orElseThrow();
        assertThat(witnessAttestation.verdict).isEqualTo(AttestationVerdict.FLAGGED);
        assertThat(witnessAttestation.confidence).isEqualTo(0.5);
    }

    @Test
    void ignoresUnmappedActionType() {
        var event = new TrustRelevantAction(
            "penelope-pitstop", "hooded-claw", "MOVE", "success",
            List.of(), "test-tenant");
        recorder.onTrustRelevantAction(event);

        assertThat(repo.savedEntries).isEmpty();
        assertThat(repo.savedAttestations).isEmpty();
    }

    @Test
    void noWitnessesProducesOnlyTargetAttestation() {
        var event = new TrustRelevantAction(
            "hooded-claw", "penelope-pitstop", "STEAL", "success",
            List.of(), "test-tenant");
        recorder.onTrustRelevantAction(event);

        assertThat(repo.savedAttestations).hasSize(1);
        assertThat(repo.savedAttestations.get(0).attestorId).isEqualTo("penelope-pitstop");
    }

    // Minimal test double — captures saves without CDI/JPA
    static class CapturingLedgerRepo implements LedgerEntryRepository {
        List<LedgerEntry> savedEntries = new ArrayList<>();
        List<LedgerAttestation> savedAttestations = new ArrayList<>();

        @Override public LedgerEntry save(LedgerEntry entry, String tenancyId) {
            if (entry.id == null) entry.id = UUID.randomUUID();
            savedEntries.add(entry); return entry;
        }
        @Override public LedgerAttestation saveAttestation(LedgerAttestation a, String tenancyId) {
            if (a.id == null) a.id = UUID.randomUUID();
            savedAttestations.add(a); return a;
        }
        // remaining interface methods — return empty defaults
        @Override public java.util.List<LedgerEntry> findBySubjectId(UUID s, String t) { return List.of(); }
        @Override public java.util.List<LedgerEntry> findBySubjectIdAndTimeRange(UUID s, java.time.Instant f, java.time.Instant to, String t) { return List.of(); }
        @Override public java.util.Optional<LedgerEntry> findLatestBySubjectId(UUID s, String t) { return java.util.Optional.empty(); }
        @Override public java.util.Optional<LedgerEntry> findEntryById(UUID i, String t) { return java.util.Optional.empty(); }
        @Override public java.util.List<LedgerAttestation> findAttestationsByEntryId(UUID e, String t) { return List.of(); }
        @Override public java.util.List<LedgerEntry> findByActorId(String a, java.time.Instant f, java.time.Instant t, String te) { return List.of(); }
        @Override public java.util.List<LedgerEntry> findByActorRole(String r, java.time.Instant f, java.time.Instant t, String te) { return List.of(); }
        @Override public java.util.List<LedgerEntry> findCausedBy(UUID e, String t) { return List.of(); }
        @Override public java.util.List<LedgerAttestation> findAttestationsByEntryIdAndCapabilityTag(UUID e, String c, String t) { return List.of(); }
        @Override public java.util.List<LedgerAttestation> findAttestationsByEntryIdGlobal(UUID e, String t) { return List.of(); }
        @Override public java.util.List<LedgerAttestation> findAttestationsByAttestorIdAndCapabilityTag(String a, String c, String t) { return List.of(); }
        @Override public java.util.stream.Stream<LedgerEntry> streamBySubjectId(UUID s, String t) { return java.util.stream.Stream.empty(); }
        @Override public java.util.stream.Stream<LedgerEntry> streamByActorId(String a, java.time.Instant f, java.time.Instant t, String te) { return java.util.stream.Stream.empty(); }
        @Override public java.util.List<LedgerEntry> findBySubjectIdPaged(UUID s, int a, int l, String t) { return List.of(); }
        @Override public java.util.Map<AttestationVerdict, Long> countByActorAndVerdict(String a, java.time.Instant f, java.time.Instant t, String te) { return java.util.Map.of(); }
        @Override public java.util.Map<AttestationVerdict, Long> countBySubjectAndVerdict(UUID s, java.time.Instant f, java.time.Instant t, String te) { return java.util.Map.of(); }
        @Override public io.casehub.ledger.api.model.AttestationSummary summariseAttestationsByActor(String a, java.time.Instant f, java.time.Instant t, String te) { return io.casehub.ledger.api.model.AttestationSummary.EMPTY; }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=TrustEventRecorderTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: compilation failure

- [ ] **Step 3: Create TrustRelevantAction record**

```java
package io.casehub.blocks.trust;

import java.util.List;
import java.util.Objects;

public record TrustRelevantAction(
    String actorId,
    String targetId,
    String actionType,
    String actionResult,
    List<String> witnessIds,
    String tenantId
) {
    public TrustRelevantAction {
        Objects.requireNonNull(actorId);
        Objects.requireNonNull(targetId);
        Objects.requireNonNull(actionType);
        witnessIds = witnessIds != null ? List.copyOf(witnessIds) : List.of();
        Objects.requireNonNull(tenantId);
    }
}
```

- [ ] **Step 4: Create TrustEventRecorder**

```java
package io.casehub.blocks.trust;

import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.ledger.runtime.model.PlainLedgerEntry;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

import java.util.UUID;

@ApplicationScoped
public class TrustEventRecorder {

    private final TrustEvolutionConfig config;
    private final LedgerEntryRepository ledgerRepo;

    @Inject
    public TrustEventRecorder(TrustEvolutionConfig config, LedgerEntryRepository ledgerRepo) {
        this.config = config;
        this.ledgerRepo = ledgerRepo;
    }

    public void onTrustRelevantAction(@ObservesAsync TrustRelevantAction event) {
        var mapping = config.findMapping(event.actionType());
        if (mapping.isEmpty()) return;

        var m = mapping.get();
        var entry = new PlainLedgerEntry();
        entry.actorId = event.actorId();
        entry.entryType = LedgerEntryType.EVENT;
        entry.subjectId = UUID.nameUUIDFromBytes(event.targetId().getBytes());
        var saved = ledgerRepo.save(entry, event.tenantId());

        createAttestation(saved.id, event.targetId(), m.verdict(), m.confidence(), event.tenantId());

        for (String witnessId : event.witnessIds()) {
            createAttestation(saved.id, witnessId, m.verdict(), m.witnessConfidence(), event.tenantId());
        }
    }

    private void createAttestation(UUID entryId, String attestorId,
                                   AttestationVerdict verdict, double confidence,
                                   String tenantId) {
        var attestation = new LedgerAttestation();
        attestation.ledgerEntryId = entryId;
        attestation.attestorId = attestorId;
        attestation.verdict = verdict;
        attestation.confidence = confidence;
        ledgerRepo.saveAttestation(attestation, tenantId);
    }
}
```

- [ ] **Step 5: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl blocks -s slot-settings.xml -Dtest=TrustEventRecorderTest`
Expected: PASS

- [ ] **Step 6: Verify with ide_diagnostics on blocks**

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add blocks/src/main/java/io/casehub/blocks/trust/ blocks/src/test/java/io/casehub/blocks/trust/
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#65): TrustRelevantAction CDI event + TrustEventRecorder observer

Refs casehubio/examples#65

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: TrustEvolutionConfigSpec in blocks-agentic-yaml

**Files:**
- Create: `agentic-yaml/src/main/java/io/casehub/blocks/agentic/yaml/spec/cognition/TrustEvolutionConfigSpec.java`
- Test: `agentic-yaml/src/test/java/io/casehub/blocks/agentic/yaml/spec/cognition/TrustEvolutionConfigSpecTest.java`

**Interfaces:**
- Consumes: nothing from other tasks
- Produces: `TrustEvolutionConfigSpec` YAML-deserializable record (consumed by DSL migration follow-up #74)

- [ ] **Step 1: Write failing test for YAML round-trip**

```java
package io.casehub.blocks.agentic.yaml.spec.cognition;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class TrustEvolutionConfigSpecTest {

    private static final ObjectMapper YAML = new ObjectMapper(new YAMLFactory());

    @Test
    void deserializesFromYaml() throws Exception {
        String yaml = """
            scoring:
              decayHalfLifeDays: 30
              negativeDecayMultiplier: 1.5
            consolidation:
              overlayProperty: trust-score
              significantChangeThreshold: 0.15
            levels:
              high: 0.7
              moderate: 0.4
              low: 0.2
            events:
              - actionType: STEAL
                verdict: FLAGGED
                confidence: 0.9
                witnessConfidence: 0.5
            """;
        var spec = YAML.readValue(yaml, TrustEvolutionConfigSpec.class);
        assertThat(spec.events()).hasSize(1);
        assertThat(spec.events().get(0).actionType()).isEqualTo("STEAL");
        assertThat(spec.scoring().decayHalfLifeDays()).isEqualTo(30);
        assertThat(spec.levels().high()).isEqualTo(0.7);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

- [ ] **Step 3: Create TrustEvolutionConfigSpec**

Following the `DriveConfigSpec` pattern (a record in the cognition package):

```java
package io.casehub.blocks.agentic.yaml.spec.cognition;

import org.jspecify.annotations.Nullable;
import java.util.List;

public record TrustEvolutionConfigSpec(
    @Nullable List<TrustEventMappingSpec> events,
    @Nullable TrustScoringSpec scoring,
    @Nullable TrustConsolidationSpec consolidation,
    @Nullable TrustLevelSpec levels
) {
    public record TrustEventMappingSpec(
        String actionType,
        String verdict,
        double confidence,
        double witnessConfidence
    ) {}

    public record TrustScoringSpec(
        int decayHalfLifeDays,
        double negativeDecayMultiplier
    ) {}

    public record TrustConsolidationSpec(
        String overlayProperty,
        double significantChangeThreshold
    ) {}

    public record TrustLevelSpec(
        double high,
        double moderate,
        double low
    ) {}
}
```

- [ ] **Step 4: Run tests, verify with ide_diagnostics**

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/blocks add agentic-yaml/src/
git -C /Users/mdproctor/claude/casehub/slots/196/blocks commit -m "feat(#65): TrustEvolutionConfigSpec in agentic-yaml cognition package

Refs casehubio/examples#65

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: Consolidation phase (neocortex-mindmap-intelligence)

### Task 4: TrustConsolidationPhase + dependencies

**Files:**
- Modify: `mindmap-intelligence/pom.xml` — add ledger-api, ledger-core, blocks-core deps
- Create: `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/TrustConsolidationPhase.java`
- Test: `mindmap-intelligence/src/test/java/io/casehub/neocortex/mindmap/intelligence/consolidation/TrustConsolidationPhaseTest.java`

**Interfaces:**
- Consumes: `TrustEvolutionConfig` (Task 1), `OverlayTrustPropertyModel` (Task 1), `TrustScoreComputer` + `DecayFunction` (ledger-core), `LedgerEntryRepository` (ledger-api), `MindMapStore` (neocortex-mindmap-api), `ConsolidationPhase` SPI
- Produces: Updated overlay node properties (`trust-score`, `trust-alpha`, `trust-beta`, `trust-last-rendered`)

- [ ] **Step 1: Add dependencies to mindmap-intelligence/pom.xml**

Add to `<dependencies>` section using Edit tool:

```xml
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-api</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-ledger-core</artifactId>
      <version>${casehub.version}</version>
    </dependency>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-blocks-core</artifactId>
      <version>${casehub.version}</version>
    </dependency>
```

- [ ] **Step 2: Write failing test for TrustConsolidationPhase**

Test that the phase reads overlay nodes, queries ledger, computes trust via TrustScoreComputer, and writes properties:

```java
package io.casehub.neocortex.mindmap.intelligence.consolidation;

import io.casehub.blocks.trust.OverlayTrustPropertyModel;
import io.casehub.blocks.trust.TrustEvolutionConfig;
import io.casehub.blocks.trust.TrustEvolutionConfig.*;
import io.casehub.ledger.api.model.AttestationVerdict;
import io.casehub.ledger.api.model.LedgerAttestation;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.runtime.model.PlainLedgerEntry;
import io.casehub.neocortex.mindmap.inmem.InMemoryMindMapStore;
import io.casehub.neocortex.mindmap.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.time.Instant;
import java.util.*;

import static org.assertj.core.api.Assertions.*;

class TrustConsolidationPhaseTest {

    // Tests verify:
    // - overlay node gets trust-score/alpha/beta properties after consolidation
    // - per-observer filtering produces different scores
    // - empty attestation history → score 0.5 (Bayesian prior)
    // - threshold gating: small changes don't update trust-last-rendered
    // - negativeDecayMultiplier applied for FLAGGED verdicts
    // (full test implementations follow TDD — write test, verify fail, implement, verify pass)
}
```

- [ ] **Step 3: Run test to verify it fails**

- [ ] **Step 4: Create TrustConsolidationPhase**

The full implementation follows the spec's algorithm (§TrustConsolidationPhase, lines 229-254). Key method structure:

```java
@ApplicationScoped
@Priority(22)
public class TrustConsolidationPhase implements ConsolidationPhase {

    private final MindMapStore mindMapStore;
    private final LedgerEntryRepository ledgerRepo;
    private final TrustEvolutionConfig config;

    @Inject
    public TrustConsolidationPhase(MindMapStore mindMapStore,
                                    LedgerEntryRepository ledgerRepo,
                                    Instance<TrustEvolutionConfig> config) {
        this.mindMapStore = mindMapStore;
        this.ledgerRepo = ledgerRepo;
        this.config = config.isResolvable() ? config.get() : null;
    }

    @Override
    public String name() { return "trust-consolidation"; }

    @Override
    public void run(String tenantId, List<String> subgraphPriority) {
        if (config == null) return;
        // 1. Find "people" subgraph
        // 2. Query overlay nodes
        // 3. For each overlay: filter attestations by observer, compute trust, update properties
    }
}
```

- [ ] **Step 5: Run tests, verify with ide_diagnostics**

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex add mindmap-intelligence/
git -C /Users/mdproctor/claude/casehub/slots/196/neocortex commit -m "feat(#65): TrustConsolidationPhase — Bayesian Beta trust during consolidation

@Priority(22), after MergeDetection. Reads ledger attestations,
computes per-relationship trust via TrustScoreComputer, writes
trust-score/alpha/beta to person-entity overlay nodes.

Refs casehubio/examples#65

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Wacky-manor integration — config + wiring

### Task 5: trust-evolution.yaml + loader + CDI producer

**Files:**
- Create: `wacky-manor/src/main/resources/META-INF/eidos/trust-evolution.yaml`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ManorTrustEvolutionConfigLoader.java`
- Create: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/TrustEvolutionConfigProducer.java`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/ManorTrustEvolutionConfigLoaderTest.java`

**Interfaces:**
- Consumes: `TrustEvolutionConfig` (Task 1)
- Produces: CDI-injectable `TrustEvolutionConfig` bean (consumed by TrustEventRecorder from Task 2, TrustConsolidationPhase from Task 4)

- [ ] **Step 1: Create trust-evolution.yaml**

Write to `wacky-manor/src/main/resources/META-INF/eidos/trust-evolution.yaml`:

```yaml
scoring:
  decay-half-life-days: 30
  negative-decay-multiplier: 1.5
consolidation:
  overlay-property: trust-score
  significant-change-threshold: 0.15
levels:
  high: 0.7
  moderate: 0.4
  low: 0.2
events:
  - type: STEAL
    verdict: FLAGGED
    confidence: 0.9
    witness-confidence: 0.5
  - type: GIVE
    verdict: SOUND
    confidence: 0.7
    witness-confidence: 0.3
  - type: PULL_ASIDE
    verdict: SOUND
    confidence: 0.5
    witness-confidence: 0.2
```

- [ ] **Step 2: Write failing test for loader**

- [ ] **Step 3: Create ManorTrustEvolutionConfigLoader**

Static utility, same pattern as `ManorSocialConfigLoader`:

```java
package io.casehub.examples.manor.agent;

import com.fasterxml.jackson.databind.ObjectMapper;
import com.fasterxml.jackson.dataformat.yaml.YAMLFactory;
import io.casehub.blocks.trust.TrustEvolutionConfig;
import io.casehub.blocks.trust.TrustEvolutionConfig.*;
import io.casehub.ledger.api.model.AttestationVerdict;

import java.io.IOException;
import java.io.InputStream;
import java.util.List;
import java.util.Map;

public final class ManorTrustEvolutionConfigLoader {
    private static final String DEFAULT_RESOURCE = "META-INF/eidos/trust-evolution.yaml";

    private ManorTrustEvolutionConfigLoader() {}

    @SuppressWarnings("unchecked")
    public static TrustEvolutionConfig load() {
        // Parse YAML → TrustEvolutionConfig record
        // Map verdict strings to AttestationVerdict enum
    }
}
```

- [ ] **Step 4: Create TrustEvolutionConfigProducer**

```java
@ApplicationScoped
public class TrustEvolutionConfigProducer {
    @Produces @ApplicationScoped
    TrustEvolutionConfig produce() {
        return ManorTrustEvolutionConfigLoader.load();
    }
}
```

- [ ] **Step 5: Run tests, verify**

- [ ] **Step 6: Commit**

### Task 6: ScenarioOrchestrator wiring + observation rendering

**Files:**
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/ScenarioOrchestrator.java` — inject `Event<TrustRelevantAction>`, fire async CDI event after action resolution, remove `ManorTrustEvents` usage
- Modify: `wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterCognition.java` — add MindMapStore + TrustEvolutionConfig constructor params, add trust rendering in `renderCognitiveSections()`
- Test: `wacky-manor/src/test/java/io/casehub/examples/manor/agent/TrustEvolutionIntegrationTest.java`

**Interfaces:**
- Consumes: `TrustRelevantAction` (Task 2), `TrustEvolutionConfig` (Task 5 producer), `OverlayTrustPropertyModel` (Task 1), `TrustSummary`/`TrustLevel`/`CognitiveObservationSections.trustSection()` (existing blocks types)
- Produces: End-to-end trust evolution — action → ledger → consolidation → overlay → observation

- [ ] **Step 1: Inject CDI event in ScenarioOrchestrator**

Use `ide_insert_member` to add field:
```java
@Inject Event<TrustRelevantAction> trustEvent;
```

- [ ] **Step 2: Wire CDI event firing after action resolution**

In `runAutonomousTicks()`, after `extractTargetAgent()`:
```java
String trustTarget = extractTargetAgent(response);
if (trustTarget != null) {
    List<String> witnessIds = world.charactersInRoom(c.currentRoom()).stream()
        .map(CharacterState::agentId)
        .filter(id -> !id.equals(c.agentId()) && !id.equals(trustTarget))
        .toList();
    trustEvent.fireAsync(new TrustRelevantAction(
        c.agentId(), trustTarget, response.action().type().name(),
        result.text(), witnessIds, tenancyId));
}
```

- [ ] **Step 3: Remove ManorTrustEvents usage from CharacterCognition.recordTrustEvent()**

Use `ide_refactor_safe_delete` if the method has no other callers. Otherwise use `ide_replace_member` to make it a no-op or remove the call site.

- [ ] **Step 4: Add trust rendering to CharacterCognition.renderCognitiveSections()**

Add `MindMapStore` and `TrustEvolutionConfig` as constructor parameters. Add trust section rendering:
```java
// In renderCognitiveSections():
List<TrustSummary> trustSummaries = buildTrustSummaries(agentId, tenantId);
if (!trustSummaries.isEmpty()) {
    sections.add(CognitiveObservationSections.trustSection(trustSummaries));
}
```

- [ ] **Step 5: Write integration test**

End-to-end: fire TrustRelevantAction → verify ledger entry created → run consolidation → verify overlay node updated → verify observation rendering produces TrustSummary.

- [ ] **Step 6: Run full wacky-manor test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -s slot-settings.xml`
Expected: all tests pass

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/196/examples add wacky-manor/
git -C /Users/mdproctor/claude/casehub/slots/196/examples commit -m "feat(#65): wire trust evolution — CDI events, consolidation, observation rendering

ScenarioOrchestrator fires TrustRelevantAction on trust-relevant actions.
TrustConsolidationPhase writes Bayesian Beta scores to overlay nodes.
CharacterCognition renders trust via CognitiveObservationSections.trustSection().
ManorTrustEvents weight-based model superseded.

Closes #65

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- `specs/issue-65-trust-evolution/2026-09-16-trust-evolution-design.md` — design spec this plan implements
- `specs/issue-65-trust-evolution/decisions.md` — D1-D14 design decisions
- `io.casehub.ledger.core.trust.TrustScoreComputer` — Bayesian Beta computation
- `io.casehub.ledger.core.trust.DecayFunction` — temporal decay SPI
- `io.casehub.ledger.runtime.model.PlainLedgerEntry` — concrete LedgerEntry subclass
- `io.casehub.ledger.memory.InMemoryLedgerEntryRepository` — in-memory ledger store
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ConsolidationPhase` — consolidation SPI
- `io.casehub.neocortex.mindmap.intelligence.consolidation.ExperienceConsolidationPhase` — reference phase implementation
- `io.casehub.blocks.summarisation.observation.affordance.CognitiveObservationSections:130` — trustSection()
- `io.casehub.blocks.summarisation.observation.affordance.TrustLevel` — HIGH/MODERATE/LOW/UNKNOWN
- `io.casehub.blocks.summarisation.observation.affordance.TrustSummary` — trust observation record
- `io.casehub.blocks.agentic.yaml.spec.cognition.DriveConfigSpec` — YAML spec pattern reference
- `io.casehub.examples.manor.agent.ManorTrustEvents` — superseded weight-based model
- `io.casehub.examples.manor.agent.ManorCognitiveSeeder:22-72` — overlay node creation pattern
- `io.casehub.examples.manor.agent.ManorSocialConfigLoader` — YAML loader pattern reference
- casehubio/examples#65 — focal issue
- casehubio/examples#64 — parent epic (Phase C)
- GE-20260429-42fb02 — Bayesian Beta trust returns 0.5 for no evidence
