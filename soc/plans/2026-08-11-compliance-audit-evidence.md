# Compliance & Audit Evidence Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #24 — Layer 5: Compliance & audit evidence
**Issue group:** #21, #22, #23, #24

**Goal:** Create a tamper-evident audit trail for SOC incident investigations with Merkle inclusion proofs, DORA response time reports, and PII sanitisation.

**Architecture:** Single `SocLedgerEntry` extends `JpaLedgerEntry` (JOINED inheritance), differentiated by `SocStepType` enum. Write path splits into two observers (CDI for intermediate steps, `CaseOutcomeObserver` SPI for resolution) sharing a `SocLedgerEntryWriter`. Read path exposes compliance endpoints via `SocComplianceResource` delegating to `SocComplianceService`.

**Tech Stack:** Java 21, Quarkus 3.32.2, JPA (JOINED inheritance), Flyway, JAX-RS, AssertJ

## Global Constraints

- Java 21 source on Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
- Test scoping: `-pl <module> -am -Dsurefire.failIfNoSpecifiedTests=false`
- All ledger entry persistence via `LedgerEntryRepository.save()` — never `em.persist()` (GE-20260612-17c161)
- Caller manages: `subjectId`, `sequenceNumber`, `entryType`, `actorId`, `actorRole` (GE-20260511-b6f903)
- Hash chain disabled in H2 tests: `casehub.ledger.hash-chain.enabled=false` (PP-20260604-f45c95)
- SOC Flyway version range: V5000+ at `db/soc/migration/` on qhorus datasource
- Use IntelliJ MCP (`mcp__intellij-index__*`) for all code navigation and editing
- Commits reference issue: `Refs #24`

---

### Task 1: Domain types — SocStepType, SocPreferences, DoraResponseTimeReport

**Files:**
- Create: `api/src/main/java/io/casehub/soc/domain/SocStepType.java`
- Create: `api/src/main/java/io/casehub/soc/domain/SocPreferences.java`
- Create: `api/src/main/java/io/casehub/soc/domain/DoraResponseTimeReport.java`
- Create: `api/src/main/java/io/casehub/soc/domain/PriorityStats.java`
- Test: `api/src/test/java/io/casehub/soc/domain/SocStepTypeTest.java`
- Test: `api/src/test/java/io/casehub/soc/domain/SocPreferencesTest.java`
- Test: `api/src/test/java/io/casehub/soc/domain/DoraResponseTimeReportTest.java`

**Interfaces:**
- Consumes: `io.casehub.platform.api.preferences.PreferenceKey`, `io.casehub.platform.api.preferences.DurationPreference`
- Produces: `SocStepType` enum (6 values), `SocPreferences` (4 `PreferenceKey<DurationPreference>` constants), `DoraResponseTimeReport` record, `PriorityStats` record

- [ ] **Step 1: Write failing test for SocStepType**

```java
package io.casehub.soc.domain;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SocStepTypeTest {

    @Test
    void allStepTypesDefined() {
        assertThat(SocStepType.values()).containsExactly(
                SocStepType.ALERT_TRIAGE,
                SocStepType.INCIDENT_PROMOTED,
                SocStepType.INVESTIGATION_STEP,
                SocStepType.CONTAINMENT_DECISION,
                SocStepType.CONTAINMENT_EXECUTED,
                SocStepType.INCIDENT_RESOLVED);
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dtest=SocStepTypeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `SocStepType` not found

- [ ] **Step 3: Implement SocStepType**

```java
package io.casehub.soc.domain;

public enum SocStepType {
    ALERT_TRIAGE,
    INCIDENT_PROMOTED,
    INVESTIGATION_STEP,
    CONTAINMENT_DECISION,
    CONTAINMENT_EXECUTED,
    INCIDENT_RESOLVED
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dtest=SocStepTypeTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Write failing test for SocPreferences**

```java
package io.casehub.soc.domain;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import static org.assertj.core.api.Assertions.assertThat;

class SocPreferencesTest {

    @Test
    void p1ResponseWindow_defaults15Minutes() {
        assertThat(SocPreferences.P1_RESPONSE_WINDOW.defaultValue().value())
                .isEqualTo(Duration.ofMinutes(15));
    }

    @Test
    void p2ResponseWindow_defaults1Hour() {
        assertThat(SocPreferences.P2_RESPONSE_WINDOW.defaultValue().value())
                .isEqualTo(Duration.ofHours(1));
    }

    @Test
    void p3ResponseWindow_defaults4Hours() {
        assertThat(SocPreferences.P3_RESPONSE_WINDOW.defaultValue().value())
                .isEqualTo(Duration.ofHours(4));
    }

    @Test
    void p4ResponseWindow_defaults24Hours() {
        assertThat(SocPreferences.P4_RESPONSE_WINDOW.defaultValue().value())
                .isEqualTo(Duration.ofHours(24));
    }

    @Test
    void allKeysHaveSocNamespace() {
        assertThat(SocPreferences.P1_RESPONSE_WINDOW.namespace()).isEqualTo("soc");
        assertThat(SocPreferences.P2_RESPONSE_WINDOW.namespace()).isEqualTo("soc");
        assertThat(SocPreferences.P3_RESPONSE_WINDOW.namespace()).isEqualTo("soc");
        assertThat(SocPreferences.P4_RESPONSE_WINDOW.namespace()).isEqualTo("soc");
    }
}
```

- [ ] **Step 6: Implement SocPreferences**

```java
package io.casehub.soc.domain;

import io.casehub.platform.api.preferences.DurationPreference;
import io.casehub.platform.api.preferences.PreferenceKey;
import java.time.Duration;

public final class SocPreferences {
    public static final PreferenceKey<DurationPreference> P1_RESPONSE_WINDOW =
        new PreferenceKey<>("soc", "p1ResponseWindow",
            DurationPreference.of(Duration.ofMinutes(15)), DurationPreference::parse);
    public static final PreferenceKey<DurationPreference> P2_RESPONSE_WINDOW =
        new PreferenceKey<>("soc", "p2ResponseWindow",
            DurationPreference.of(Duration.ofHours(1)), DurationPreference::parse);
    public static final PreferenceKey<DurationPreference> P3_RESPONSE_WINDOW =
        new PreferenceKey<>("soc", "p3ResponseWindow",
            DurationPreference.of(Duration.ofHours(4)), DurationPreference::parse);
    public static final PreferenceKey<DurationPreference> P4_RESPONSE_WINDOW =
        new PreferenceKey<>("soc", "p4ResponseWindow",
            DurationPreference.of(Duration.ofHours(24)), DurationPreference::parse);
    private SocPreferences() {}
}
```

- [ ] **Step 7: Run SocPreferencesTest**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dtest=SocPreferencesTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Write failing test for DoraResponseTimeReport**

```java
package io.casehub.soc.domain;

import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.Map;
import static org.assertj.core.api.Assertions.assertThat;

class DoraResponseTimeReportTest {

    @Test
    void emptyReport_zeroIncidents() {
        var report = new DoraResponseTimeReport(
                Instant.parse("2026-08-01T00:00:00Z"),
                Instant.parse("2026-08-31T23:59:59Z"),
                0, Map.of());
        assertThat(report.totalIncidents()).isEqualTo(0);
        assertThat(report.byPriority()).isEmpty();
    }

    @Test
    void priorityStats_computesSlaCompliance() {
        var stats = new PriorityStats(10, Duration.ofMinutes(5),
                Duration.ofMinutes(30), Duration.ofHours(2), 0.9);
        assertThat(stats.count()).isEqualTo(10);
        assertThat(stats.avgTimeToTriage()).isEqualTo(Duration.ofMinutes(5));
        assertThat(stats.slaCompliancePercent()).isEqualTo(0.9);
    }
}
```

- [ ] **Step 9: Implement DoraResponseTimeReport and PriorityStats**

```java
// DoraResponseTimeReport.java
package io.casehub.soc.domain;

import java.time.Instant;
import java.util.Map;

public record DoraResponseTimeReport(
    Instant reportPeriodStart,
    Instant reportPeriodEnd,
    int totalIncidents,
    Map<String, PriorityStats> byPriority) {}
```

```java
// PriorityStats.java
package io.casehub.soc.domain;

import java.time.Duration;

public record PriorityStats(
    int count,
    Duration avgTimeToTriage,
    Duration avgTimeToContainment,
    Duration avgTimeToResolution,
    double slaCompliancePercent) {}
```

- [ ] **Step 10: Run all api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl api -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 11: Commit**

```bash
git add api/src/main/java/io/casehub/soc/domain/SocStepType.java api/src/main/java/io/casehub/soc/domain/SocPreferences.java api/src/main/java/io/casehub/soc/domain/DoraResponseTimeReport.java api/src/main/java/io/casehub/soc/domain/PriorityStats.java api/src/test/java/io/casehub/soc/domain/SocStepTypeTest.java api/src/test/java/io/casehub/soc/domain/SocPreferencesTest.java api/src/test/java/io/casehub/soc/domain/DoraResponseTimeReportTest.java
git commit -m "feat(#24): add SocStepType, SocPreferences, DoraResponseTimeReport

Refs #24"
```

---

### Task 2: SocLedgerEntry entity + Flyway migration + config

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntry.java`
- Create: `app/src/main/resources/db/soc/migration/V5000__soc_ledger_entry.sql`
- Modify: `app/src/main/resources/application.properties` — add `db/soc/migration` to qhorus flyway locations
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocLedgerEntryTest.java`

**Interfaces:**
- Consumes: `SocStepType` (Task 1), `io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry`
- Produces: `SocLedgerEntry` class — `incidentId` (UUID), `stepType` (SocStepType), `domainContentBytes()`, `@PrePersist` invariant enforcement

- [ ] **Step 1: Write failing test for SocLedgerEntry**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.soc.domain.SocStepType;
import org.junit.jupiter.api.Test;
import java.nio.charset.StandardCharsets;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SocLedgerEntryTest {

    @Test
    void domainContentBytes_includesIncidentIdAndStepType() {
        var entry = new SocLedgerEntry();
        entry.incidentId = UUID.fromString("11111111-1111-1111-1111-111111111111");
        entry.stepType = SocStepType.ALERT_TRIAGE;

        String result = new String(entry.domainContentBytes(), StandardCharsets.UTF_8);
        assertThat(result).isEqualTo("11111111-1111-1111-1111-111111111111|ALERT_TRIAGE");
    }

    @Test
    void domainContentBytes_nullsSafelyHandled() {
        var entry = new SocLedgerEntry();
        String result = new String(entry.domainContentBytes(), StandardCharsets.UTF_8);
        assertThat(result).isEqualTo("|");
    }

    @Test
    void prePersist_throwsWhenIncidentIdDoesNotMatchSubjectId() {
        var entry = new SocLedgerEntry();
        entry.incidentId = UUID.randomUUID();
        entry.subjectId = UUID.randomUUID();
        assertThatThrownBy(entry::validateIncidentIdInvariant)
                .isInstanceOf(IllegalStateException.class);
    }

    @Test
    void prePersist_succeedsWhenIncidentIdMatchesSubjectId() {
        var entry = new SocLedgerEntry();
        UUID id = UUID.randomUUID();
        entry.incidentId = id;
        entry.subjectId = id;
        entry.validateIncidentIdInvariant();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocLedgerEntryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `SocLedgerEntry` not found

- [ ] **Step 3: Implement SocLedgerEntry**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.ledger.runtime.model.jpa.JpaLedgerEntry;
import io.casehub.soc.domain.SocStepType;
import jakarta.persistence.Column;
import jakarta.persistence.DiscriminatorValue;
import jakarta.persistence.Entity;
import jakarta.persistence.EnumType;
import jakarta.persistence.Enumerated;
import jakarta.persistence.PrePersist;
import jakarta.persistence.Table;
import java.nio.charset.StandardCharsets;
import java.util.UUID;

@Entity
@Table(name = "soc_ledger_entry")
@DiscriminatorValue("SOC")
public class SocLedgerEntry extends JpaLedgerEntry {

    @Column(name = "incident_id", nullable = false)
    public UUID incidentId;

    @Enumerated(EnumType.STRING)
    @Column(name = "step_type", nullable = false, length = 30)
    public SocStepType stepType;

    @Override
    protected byte[] domainContentBytes() {
        return String.join("|",
                incidentId != null ? incidentId.toString() : "",
                stepType != null ? stepType.name() : "")
            .getBytes(StandardCharsets.UTF_8);
    }

    @PrePersist
    void validateIncidentIdInvariant() {
        if (incidentId != null && subjectId != null && !incidentId.equals(subjectId)) {
            throw new IllegalStateException(
                "incidentId must equal subjectId: incidentId=" + incidentId + " subjectId=" + subjectId);
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocLedgerEntryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Create Flyway migration**

Create `app/src/main/resources/db/soc/migration/V5000__soc_ledger_entry.sql`:

```sql
-- V5000: soc_ledger_entry — SOC compliance audit trail
-- Extends ledger_entry (JOINED inheritance). V5000+ reserved by casehub-soc.

CREATE TABLE soc_ledger_entry (
    id            UUID         NOT NULL,
    incident_id   UUID         NOT NULL,
    step_type     VARCHAR(30)  NOT NULL,
    CONSTRAINT pk_soc_ledger_entry PRIMARY KEY (id),
    CONSTRAINT fk_soc_ledger_entry FOREIGN KEY (id) REFERENCES ledger_entry(id)
);

CREATE INDEX idx_sle_incident_id ON soc_ledger_entry (incident_id);
CREATE INDEX idx_sle_step_type ON soc_ledger_entry (step_type);
```

- [ ] **Step 6: Update application.properties — add SOC migration location to qhorus flyway**

In `app/src/main/resources/application.properties`, change:

```properties
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration
```

to:

```properties
quarkus.flyway.qhorus.locations=classpath:db/qhorus/migration,classpath:db/ledger/migration,classpath:db/soc/migration
```

- [ ] **Step 7: Run full build to verify migration and entity**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app -am`
Expected: PASS — Flyway runs V5000, Hibernate discovers SocLedgerEntry

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntry.java app/src/main/resources/db/soc/migration/V5000__soc_ledger_entry.sql app/src/main/resources/application.properties app/src/test/java/io/casehub/soc/engine/compliance/SocLedgerEntryTest.java
git commit -m "feat(#24): add SocLedgerEntry entity and Flyway migration V5000

JOINED inheritance subclass with incidentId + stepType. @PrePersist
validates incidentId == subjectId invariant. Migration creates join
table on qhorus datasource.

Refs #24"
```

---

### Task 3: SocPiiSanitiser

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocPiiSanitiser.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocPiiSanitiserTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `SocPiiSanitiser.sanitise(String json) → String` — replaces IPv4, IPv6, emails with `[REDACTED-*]`. Returns `[SANITISATION_FAILED]` on error.

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.soc.engine.compliance;

import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.params.ParameterizedTest;
import org.junit.jupiter.params.provider.CsvSource;
import static org.assertj.core.api.Assertions.assertThat;

class SocPiiSanitiserTest {

    private SocPiiSanitiser sanitiser;

    @BeforeEach
    void setUp() { sanitiser = new SocPiiSanitiser(); }

    @ParameterizedTest
    @CsvSource({
        "'10.0.0.1',          '[REDACTED-IP]'",
        "'192.168.1.100',     '[REDACTED-IP]'",
        "'255.255.255.255',   '[REDACTED-IP]'",
    })
    void ipv4_redacted(String input, String expected) {
        assertThat(sanitiser.sanitise(input)).isEqualTo(expected);
    }

    @ParameterizedTest
    @CsvSource({
        "'::1',                        '[REDACTED-IP]'",
        "'::ffff:10.0.0.1',            '[REDACTED-IP]'",
        "'2001:db8::1',                '[REDACTED-IP]'",
        "'fe80::1%25eth0',             '[REDACTED-IP]'",
    })
    void ipv6_redacted(String input, String expected) {
        assertThat(sanitiser.sanitise(input)).isEqualTo(expected);
    }

    @Test
    void email_redacted() {
        assertThat(sanitiser.sanitise("user@example.com")).isEqualTo("[REDACTED-EMAIL]");
        assertThat(sanitiser.sanitise("first.last+tag@corp.co.uk")).isEqualTo("[REDACTED-EMAIL]");
    }

    @Test
    void preserves_attckIds() {
        assertThat(sanitiser.sanitise("T1566.001")).isEqualTo("T1566.001");
    }

    @Test
    void preserves_uuids() {
        String uuid = "550e8400-e29b-41d4-a716-446655440000";
        assertThat(sanitiser.sanitise(uuid)).isEqualTo(uuid);
    }

    @Test
    void preserves_severity_values() {
        assertThat(sanitiser.sanitise("CRITICAL")).isEqualTo("CRITICAL");
        assertThat(sanitiser.sanitise("HIGH")).isEqualTo("HIGH");
    }

    @Test
    void mixed_content_redacted() {
        String input = "{\"src_ip\":\"10.0.0.1\",\"attck\":\"T1566\",\"analyst\":\"user@acme.com\"}";
        String result = sanitiser.sanitise(input);
        assertThat(result).contains("[REDACTED-IP]");
        assertThat(result).contains("[REDACTED-EMAIL]");
        assertThat(result).contains("T1566");
        assertThat(result).doesNotContain("10.0.0.1");
        assertThat(result).doesNotContain("user@acme.com");
    }

    @Test
    void nullInput_returnsSanitisationFailed() {
        assertThat(sanitiser.sanitise(null)).isEqualTo("[SANITISATION_FAILED]");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocPiiSanitiserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement SocPiiSanitiser**

```java
package io.casehub.soc.engine.compliance;

import jakarta.enterprise.context.ApplicationScoped;
import org.jboss.logging.Logger;
import java.util.regex.Pattern;

@ApplicationScoped
public class SocPiiSanitiser {

    private static final Logger LOG = Logger.getLogger(SocPiiSanitiser.class);

    private static final Pattern IPV4 = Pattern.compile(
            "\\b\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\b");

    private static final Pattern IPV6 = Pattern.compile(
            "(?i)(?:(?:[0-9a-f]{1,4}:){7}[0-9a-f]{1,4}" +
            "|(?:[0-9a-f]{1,4}:){1,7}:" +
            "|(?:[0-9a-f]{1,4}:){1,6}:[0-9a-f]{1,4}" +
            "|(?:[0-9a-f]{1,4}:){1,5}(?::[0-9a-f]{1,4}){1,2}" +
            "|(?:[0-9a-f]{1,4}:){1,4}(?::[0-9a-f]{1,4}){1,3}" +
            "|(?:[0-9a-f]{1,4}:){1,3}(?::[0-9a-f]{1,4}){1,4}" +
            "|(?:[0-9a-f]{1,4}:){1,2}(?::[0-9a-f]{1,4}){1,5}" +
            "|[0-9a-f]{1,4}:(?::[0-9a-f]{1,4}){1,6}" +
            "|:(?::[0-9a-f]{1,4}){1,7}" +
            "|::(?:[fF]{4}:)?\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}\\.\\d{1,3}" +
            "|fe80:(?::[0-9a-f]{1,4}){0,4}%[0-9a-zA-Z]+)");

    private static final Pattern EMAIL = Pattern.compile(
            "\\b[A-Za-z0-9._%+\\-]+@[A-Za-z0-9.\\-]+\\.[A-Za-z]{2,}\\b");

    private static final String REDACTED_IP = "[REDACTED-IP]";
    private static final String REDACTED_EMAIL = "[REDACTED-EMAIL]";
    private static final String SANITISATION_FAILED = "[SANITISATION_FAILED]";

    public String sanitise(String input) {
        if (input == null) {
            return SANITISATION_FAILED;
        }
        try {
            String result = IPV4.matcher(input).replaceAll(REDACTED_IP);
            result = IPV6.matcher(result).replaceAll(REDACTED_IP);
            result = EMAIL.matcher(result).replaceAll(REDACTED_EMAIL);
            return result;
        } catch (Exception e) {
            LOG.errorf(e, "PII sanitisation failed — returning SANITISATION_FAILED");
            return SANITISATION_FAILED;
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocPiiSanitiserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — adjust IPv6 regex if specific forms fail

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/compliance/SocPiiSanitiser.java app/src/test/java/io/casehub/soc/engine/compliance/SocPiiSanitiserTest.java
git commit -m "feat(#24): add SocPiiSanitiser — regex PII redaction

IPv4, IPv6, email sanitisation for compliance report metadata.
Fail-closed: returns [SANITISATION_FAILED] on error. Person name
detection deferred.

Refs #24"
```

---

### Task 4: SocLedgerEntryRepository

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntryRepository.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocLedgerEntryRepositoryTest.java`

**Interfaces:**
- Consumes: `SocLedgerEntry` (Task 2), `io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit`, `jakarta.persistence.EntityManager`
- Produces: `findByIncidentId(UUID, String) → List<SocLedgerEntry>`, `findByTimeRange(Instant, Instant, String) → List<SocLedgerEntry>`, `findByStepType(SocStepType, String) → List<SocLedgerEntry>`

- [ ] **Step 1: Write failing test**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.soc.domain.SocStepType;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.assertThat;

class SocLedgerEntryRepositoryTest {

    @Test
    void classExists_andHasExpectedMethods() throws NoSuchMethodException {
        var clazz = SocLedgerEntryRepository.class;
        assertThat(clazz.getMethod("findByIncidentId", java.util.UUID.class, String.class))
                .isNotNull();
        assertThat(clazz.getMethod("findByTimeRange", java.time.Instant.class,
                java.time.Instant.class, String.class)).isNotNull();
        assertThat(clazz.getMethod("findByStepType", SocStepType.class, String.class))
                .isNotNull();
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocLedgerEntryRepositoryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement SocLedgerEntryRepository**

Follow `CaseLedgerEntryRepository` pattern — `@DefaultBean`, `@ApplicationScoped`, `@LedgerPersistenceUnit EntityManager`.

```java
package io.casehub.soc.engine.compliance;

import io.casehub.ledger.runtime.persistence.LedgerPersistenceUnit;
import io.casehub.soc.domain.SocStepType;
import io.quarkus.arc.DefaultBean;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.persistence.EntityManager;
import jakarta.transaction.Transactional;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@DefaultBean
@ApplicationScoped
public class SocLedgerEntryRepository {

    @Inject @LedgerPersistenceUnit EntityManager em;

    @Transactional
    public List<SocLedgerEntry> findByIncidentId(UUID incidentId, String tenancyId) {
        return em.createQuery(
                "SELECT e FROM SocLedgerEntry e WHERE e.incidentId = :incidentId AND e.tenancyId = :tenancyId ORDER BY e.sequenceNumber ASC",
                SocLedgerEntry.class)
            .setParameter("incidentId", incidentId)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
    }

    @Transactional
    public List<SocLedgerEntry> findByTimeRange(Instant from, Instant to, String tenancyId) {
        return em.createQuery(
                "SELECT e FROM SocLedgerEntry e WHERE e.occurredAt >= :from AND e.occurredAt <= :to AND e.tenancyId = :tenancyId ORDER BY e.occurredAt ASC",
                SocLedgerEntry.class)
            .setParameter("from", from)
            .setParameter("to", to)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
    }

    @Transactional
    public List<SocLedgerEntry> findByStepType(SocStepType stepType, String tenancyId) {
        return em.createQuery(
                "SELECT e FROM SocLedgerEntry e WHERE e.stepType = :stepType AND e.tenancyId = :tenancyId ORDER BY e.occurredAt ASC",
                SocLedgerEntry.class)
            .setParameter("stepType", stepType)
            .setParameter("tenancyId", tenancyId)
            .getResultList();
    }
}
```

Note: the JPQL uses `SocLedgerEntry` directly — JPA polymorphic queries on the discriminator handle the join.

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocLedgerEntryRepositoryTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntryRepository.java app/src/test/java/io/casehub/soc/engine/compliance/SocLedgerEntryRepositoryTest.java
git commit -m "feat(#24): add SocLedgerEntryRepository — typed queries

findByIncidentId, findByTimeRange, findByStepType. Uses
@LedgerPersistenceUnit EntityManager following CaseLedgerEntryRepository
pattern.

Refs #24"
```

---

### Task 5: SocLedgerEntryWriter + SocResolutionLedgerObserver

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntryWriter.java`
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocResolutionLedgerObserver.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocLedgerEntryWriterTest.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocResolutionLedgerObserverTest.java`

**Interfaces:**
- Consumes: `SocLedgerEntry` (Task 2), `SocStepType` (Task 1), `LedgerEntryRepository.save()`, `LedgerEntryRepository.findLatestBySubjectId()`, `CaseOutcomeObserver`, `SocCaseOutcomeFilter`
- Produces: `SocLedgerEntryWriter.write(UUID incidentId, SocStepType, String actorId, String actorRole, ActorType actorType, String metadataJson, String tenancyId, UUID causedByEntryId)`, `SocResolutionLedgerObserver` implementing `CaseOutcomeObserver`

- [ ] **Step 1: Write failing test for SocLedgerEntryWriter**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.soc.domain.SocStepType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.Optional;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class SocLedgerEntryWriterTest {

    private CapturingLedgerRepo repo;
    private SocLedgerEntryWriter writer;
    private static final Clock FIXED_CLOCK = Clock.fixed(
            Instant.parse("2026-08-11T10:00:00Z"), ZoneOffset.UTC);

    @BeforeEach
    void setUp() {
        repo = new CapturingLedgerRepo();
        writer = new SocLedgerEntryWriter(repo, FIXED_CLOCK);
    }

    @Test
    void write_setsAllRequiredFields() {
        UUID incidentId = UUID.randomUUID();
        UUID causedBy = UUID.randomUUID();
        writer.write(incidentId, SocStepType.ALERT_TRIAGE,
                "agent-1", "triage-agent", ActorType.AGENT,
                "{\"alertSeverity\":\"CRITICAL\",\"assignedSeverity\":\"HIGH\",\"triageAgentId\":\"agent-1\"}",
                "tenant-1", causedBy);

        assertThat(repo.lastSaved).isNotNull();
        SocLedgerEntry entry = (SocLedgerEntry) repo.lastSaved;
        assertThat(entry.incidentId).isEqualTo(incidentId);
        assertThat(entry.subjectId).isEqualTo(incidentId);
        assertThat(entry.stepType).isEqualTo(SocStepType.ALERT_TRIAGE);
        assertThat(entry.actorId).isEqualTo("agent-1");
        assertThat(entry.actorRole).isEqualTo("triage-agent");
        assertThat(entry.actorType).isEqualTo(ActorType.AGENT);
        assertThat(entry.entryType).isEqualTo(LedgerEntryType.EVENT);
        assertThat(entry.occurredAt).isEqualTo(FIXED_CLOCK.instant());
        assertThat(entry.causedByEntryId).isEqualTo(causedBy);
        assertThat(entry.metadata).contains("alertSeverity");
        assertThat(repo.lastTenancyId).isEqualTo("tenant-1");
    }

    @Test
    void write_sequenceNumberIncrements() {
        UUID incidentId = UUID.randomUUID();
        repo.latestSequence = 3;
        writer.write(incidentId, SocStepType.INVESTIGATION_STEP,
                "worker-1", "investigator", ActorType.AGENT,
                "{\"capabilityTag\":\"ioc\",\"investigationType\":\"ioc-enrichment\"}",
                "tenant-1", null);

        assertThat(((SocLedgerEntry) repo.lastSaved).sequenceNumber).isEqualTo(4);
    }

    @Test
    void write_containmentDecision_missingApproverId_throws() {
        UUID incidentId = UUID.randomUUID();
        assertThatThrownBy(() -> writer.write(incidentId,
                SocStepType.CONTAINMENT_DECISION,
                "agent-1", "containment", ActorType.AGENT,
                "{\"riskClassification\":\"HIGH\",\"containmentAction\":\"ISOLATE\"}",
                "tenant-1", null))
            .isInstanceOf(IllegalStateException.class)
            .hasMessageContaining("approverId");
    }

    @Test
    void write_containmentDecision_allFieldsPresent_succeeds() {
        UUID incidentId = UUID.randomUUID();
        writer.write(incidentId, SocStepType.CONTAINMENT_DECISION,
                "agent-1", "containment", ActorType.AGENT,
                "{\"approverId\":\"analyst-1\",\"riskClassification\":\"HIGH\",\"containmentAction\":\"ISOLATE\"}",
                "tenant-1", null);
        assertThat(repo.lastSaved).isNotNull();
    }

    static class CapturingLedgerRepo extends io.casehub.soc.engine.SocAttestationServiceTest.StubLedgerEntryRepository {
        LedgerEntry lastSaved;
        String lastTenancyId;
        int latestSequence = 0;

        @Override
        public LedgerEntry save(LedgerEntry entry, String tenancyId) {
            lastSaved = entry;
            lastTenancyId = tenancyId;
            return entry;
        }

        @Override
        public Optional<LedgerEntry> findLatestBySubjectId(UUID subjectId, String tenancyId) {
            if (latestSequence == 0) return Optional.empty();
            SocLedgerEntry e = new SocLedgerEntry();
            e.sequenceNumber = latestSequence;
            return Optional.of(e);
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocLedgerEntryWriterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement SocLedgerEntryWriter**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.ledger.api.model.LedgerEntryType;
import io.casehub.ledger.api.spi.LedgerEntryRepository;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.soc.domain.SocStepType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.time.Clock;
import java.util.Map;
import java.util.Set;
import java.util.UUID;

@ApplicationScoped
public class SocLedgerEntryWriter {

    private static final Logger LOG = Logger.getLogger(SocLedgerEntryWriter.class);

    private static final Map<SocStepType, Set<String>> REQUIRED_METADATA = Map.of(
        SocStepType.ALERT_TRIAGE, Set.of("alertSeverity", "assignedSeverity", "triageAgentId"),
        SocStepType.INCIDENT_PROMOTED, Set.of("promotionReason"),
        SocStepType.INVESTIGATION_STEP, Set.of("capabilityTag", "investigationType"),
        SocStepType.CONTAINMENT_DECISION, Set.of("approverId", "riskClassification", "containmentAction"),
        SocStepType.CONTAINMENT_EXECUTED, Set.of("executionResult", "containmentAction"),
        SocStepType.INCIDENT_RESOLVED, Set.of("resolutionOutcome")
    );

    private final LedgerEntryRepository ledgerRepo;
    private final Clock clock;

    @Inject
    SocLedgerEntryWriter(LedgerEntryRepository ledgerRepo, Clock clock) {
        this.ledgerRepo = ledgerRepo;
        this.clock = clock;
    }

    public void write(UUID incidentId, SocStepType stepType, String actorId,
                      String actorRole, ActorType actorType, String metadataJson,
                      String tenancyId, UUID causedByEntryId) {
        validateMetadata(stepType, metadataJson);

        SocLedgerEntry entry = new SocLedgerEntry();
        entry.incidentId = incidentId;
        entry.subjectId = incidentId;
        entry.stepType = stepType;
        entry.entryType = LedgerEntryType.EVENT;
        entry.actorId = actorId;
        entry.actorRole = actorRole;
        entry.actorType = actorType;
        entry.occurredAt = clock.instant();
        entry.metadata = metadataJson;
        entry.causedByEntryId = causedByEntryId;

        int nextSeq = ledgerRepo.findLatestBySubjectId(incidentId, tenancyId)
                .map(e -> e.sequenceNumber + 1)
                .orElse(1);
        entry.sequenceNumber = nextSeq;

        ledgerRepo.save(entry, tenancyId);
    }

    private void validateMetadata(SocStepType stepType, String metadataJson) {
        Set<String> required = REQUIRED_METADATA.getOrDefault(stepType, Set.of());
        for (String field : required) {
            if (metadataJson == null || !metadataJson.contains("\"" + field + "\"")) {
                throw new IllegalStateException(
                    "Missing required metadata field '" + field + "' for step type " + stepType);
            }
        }
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocLedgerEntryWriterTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Write failing test for SocResolutionLedgerObserver**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.soc.domain.SocCaseTypes;
import io.casehub.soc.domain.SocStepType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.Map;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class SocResolutionLedgerObserverTest {

    private SocLedgerEntryWriterTest.CapturingLedgerRepo repo;
    private SocResolutionLedgerObserver observer;

    @BeforeEach
    void setUp() {
        repo = new SocLedgerEntryWriterTest.CapturingLedgerRepo();
        var writer = new SocLedgerEntryWriter(repo,
                java.time.Clock.fixed(Instant.parse("2026-08-11T12:00:00Z"), java.time.ZoneOffset.UTC));
        observer = new SocResolutionLedgerObserver(writer);
    }

    @Test
    void onOutcome_successfulInvestigation_writesResolvedEntry() {
        UUID caseId = UUID.randomUUID();
        observer.onOutcome(new CaseOutcomeEvent(
                SocCaseTypes.INCIDENT_INVESTIGATION, "tenant-1", caseId,
                Map.of("analystOutcome", "CONFIRM_SEVERITY", "analystId", "analyst-1"),
                "resolved", Instant.now(), Map.of()));

        assertThat(repo.lastSaved).isNotNull();
        SocLedgerEntry entry = (SocLedgerEntry) repo.lastSaved;
        assertThat(entry.stepType).isEqualTo(SocStepType.INCIDENT_RESOLVED);
        assertThat(entry.incidentId).isEqualTo(caseId);
        assertThat(entry.metadata).contains("resolutionOutcome");
    }

    @Test
    void onOutcome_nonSocCase_skips() {
        observer.onOutcome(new CaseOutcomeEvent(
                "aml-investigation", "tenant-1", UUID.randomUUID(),
                Map.of(), "resolved", Instant.now(), Map.of()));
        assertThat(repo.lastSaved).isNull();
    }

    @Test
    void onOutcome_failedOutcome_skips() {
        observer.onOutcome(new CaseOutcomeEvent(
                SocCaseTypes.INCIDENT_INVESTIGATION, "tenant-1", UUID.randomUUID(),
                Map.of(), "FAULTED", Instant.now(), Map.of()));
        assertThat(repo.lastSaved).isNull();
    }
}
```

- [ ] **Step 6: Implement SocResolutionLedgerObserver**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.api.spi.CaseOutcomeEvent;
import io.casehub.api.spi.CaseOutcomeObserver;
import io.casehub.soc.domain.SocStepType;
import io.casehub.soc.engine.cbr.SocCaseOutcomeFilter;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import java.util.Map;

@ApplicationScoped
public class SocResolutionLedgerObserver implements CaseOutcomeObserver {

    private final SocLedgerEntryWriter writer;

    @Inject
    SocResolutionLedgerObserver(SocLedgerEntryWriter writer) {
        this.writer = writer;
    }

    @Override
    public void onOutcome(CaseOutcomeEvent event) {
        if (!SocCaseOutcomeFilter.isSuccessfulIncidentInvestigation(event)) {
            return;
        }

        Map<String, Object> snapshot = event.caseFileSnapshot();
        String outcome = snapshot.getOrDefault("analystOutcome", event.outcomeLabel()).toString();
        String analystId = snapshot.getOrDefault("analystId", "system:soc-compliance").toString();

        String metadataJson = "{\"resolutionOutcome\":\"" + sanitiseJsonValue(outcome) + "\"}";

        writer.write(event.caseId(), SocStepType.INCIDENT_RESOLVED,
                analystId, "incident-resolution",
                "system:soc-compliance".equals(analystId) ? ActorType.SYSTEM : ActorType.HUMAN,
                metadataJson, event.tenancyId(), null);
    }

    private static String sanitiseJsonValue(String value) {
        if (value == null) return "";
        return value.replace("\\", "\\\\").replace("\"", "\\\"");
    }
}
```

- [ ] **Step 7: Run both tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest="SocLedgerEntryWriterTest,SocResolutionLedgerObserverTest" -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/compliance/SocLedgerEntryWriter.java app/src/main/java/io/casehub/soc/engine/compliance/SocResolutionLedgerObserver.java app/src/test/java/io/casehub/soc/engine/compliance/SocLedgerEntryWriterTest.java app/src/test/java/io/casehub/soc/engine/compliance/SocResolutionLedgerObserverTest.java
git commit -m "feat(#24): add SocLedgerEntryWriter and SocResolutionLedgerObserver

Shared write helper validates compliance-critical metadata fields per
step type. Resolution observer implements CaseOutcomeObserver SPI for
INCIDENT_RESOLVED entries.

Refs #24"
```

---

### Task 6: SocComplianceService + SocComplianceResource

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocComplianceService.java`
- Create: `app/src/main/java/io/casehub/soc/rest/SocComplianceResource.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocComplianceServiceTest.java`

**Interfaces:**
- Consumes: `SocLedgerEntryRepository` (Task 4), `SocPiiSanitiser` (Task 3), `LedgerVerificationService`, `SocPreferences` (Task 1), `CurrentPrincipal`
- Produces: REST endpoints at `/api/soc/compliance/proof/{entryId}`, `/api/soc/compliance/timeline/{incidentId}`, `/api/soc/compliance/dora`

- [ ] **Step 1: Write failing test for SocComplianceService DORA aggregation**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.soc.domain.DoraResponseTimeReport;
import io.casehub.soc.domain.SocStepType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Duration;
import java.time.Instant;
import java.util.List;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class SocComplianceServiceTest {

    private SocComplianceService service;
    private StubSocLedgerEntryRepository repo;
    private SocPiiSanitiser sanitiser;

    @BeforeEach
    void setUp() {
        repo = new StubSocLedgerEntryRepository();
        sanitiser = new SocPiiSanitiser();
        service = new SocComplianceService(null, repo, sanitiser);
    }

    @Test
    void doraReport_singleIncident_computesDurations() {
        UUID incidentId = UUID.randomUUID();
        Instant base = Instant.parse("2026-08-01T10:00:00Z");

        repo.entries = List.of(
            socEntry(incidentId, SocStepType.ALERT_TRIAGE, base,
                    "{\"alertSeverity\":\"CRITICAL\",\"assignedSeverity\":\"CRITICAL\",\"triageAgentId\":\"a1\"}", 1),
            socEntry(incidentId, SocStepType.CONTAINMENT_DECISION, base.plus(Duration.ofMinutes(5)),
                    "{\"approverId\":\"analyst-1\",\"riskClassification\":\"HIGH\",\"containmentAction\":\"ISOLATE\"}", 2),
            socEntry(incidentId, SocStepType.INCIDENT_RESOLVED, base.plus(Duration.ofMinutes(10)),
                    "{\"resolutionOutcome\":\"CONFIRM_SEVERITY\"}", 3)
        );

        DoraResponseTimeReport report = service.doraReport(
                base.minus(Duration.ofHours(1)), base.plus(Duration.ofHours(1)), "tenant-1");

        assertThat(report.totalIncidents()).isEqualTo(1);
        assertThat(report.byPriority()).containsKey("CRITICAL");
        var stats = report.byPriority().get("CRITICAL");
        assertThat(stats.count()).isEqualTo(1);
        assertThat(stats.avgTimeToResolution()).isEqualTo(Duration.ofMinutes(10));
    }

    @Test
    void doraReport_noData_returnsEmptyReport() {
        repo.entries = List.of();
        Instant from = Instant.parse("2026-08-01T00:00:00Z");
        Instant to = Instant.parse("2026-08-31T23:59:59Z");

        DoraResponseTimeReport report = service.doraReport(from, to, "tenant-1");

        assertThat(report.totalIncidents()).isEqualTo(0);
        assertThat(report.byPriority()).isEmpty();
    }

    @Test
    void incidentTimeline_sanitisesPii() {
        UUID incidentId = UUID.randomUUID();
        repo.timelineEntries = List.of(
            socEntry(incidentId, SocStepType.ALERT_TRIAGE, Instant.now(),
                    "{\"src_ip\":\"10.0.0.1\",\"alertSeverity\":\"HIGH\",\"assignedSeverity\":\"HIGH\",\"triageAgentId\":\"a1\"}", 1)
        );

        List<SocLedgerEntry> timeline = service.incidentTimeline(incidentId, "tenant-1");

        assertThat(timeline).hasSize(1);
        assertThat(timeline.getFirst().metadata).contains("[REDACTED-IP]");
        assertThat(timeline.getFirst().metadata).doesNotContain("10.0.0.1");
    }

    private SocLedgerEntry socEntry(UUID incidentId, SocStepType type,
            Instant occurredAt, String metadata, int seq) {
        SocLedgerEntry e = new SocLedgerEntry();
        e.incidentId = incidentId;
        e.subjectId = incidentId;
        e.stepType = type;
        e.occurredAt = occurredAt;
        e.metadata = metadata;
        e.sequenceNumber = seq;
        e.tenancyId = "tenant-1";
        return e;
    }

    static class StubSocLedgerEntryRepository extends SocLedgerEntryRepository {
        List<SocLedgerEntry> entries = List.of();
        List<SocLedgerEntry> timelineEntries = List.of();

        @Override
        public List<SocLedgerEntry> findByTimeRange(Instant from, Instant to, String tenancyId) {
            return entries;
        }

        @Override
        public List<SocLedgerEntry> findByIncidentId(UUID incidentId, String tenancyId) {
            return timelineEntries;
        }
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocComplianceServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL

- [ ] **Step 3: Implement SocComplianceService**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.ledger.runtime.service.LedgerVerificationService;
import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.soc.domain.DoraResponseTimeReport;
import io.casehub.soc.domain.PriorityStats;
import io.casehub.soc.domain.SocPreferences;
import io.casehub.soc.domain.SocStepType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import org.jboss.logging.Logger;
import java.time.Duration;
import java.time.Instant;
import java.util.ArrayList;
import java.util.HashMap;
import java.util.List;
import java.util.Map;
import java.util.UUID;
import java.util.stream.Collectors;

@ApplicationScoped
public class SocComplianceService {

    private static final Logger LOG = Logger.getLogger(SocComplianceService.class);
    private static final Map<String, Duration> SLA_WINDOWS = Map.of(
        "CRITICAL", SocPreferences.P1_RESPONSE_WINDOW.defaultValue().value(),
        "HIGH", SocPreferences.P2_RESPONSE_WINDOW.defaultValue().value(),
        "MEDIUM", SocPreferences.P3_RESPONSE_WINDOW.defaultValue().value(),
        "LOW", SocPreferences.P4_RESPONSE_WINDOW.defaultValue().value()
    );

    private final LedgerVerificationService verificationService;
    private final SocLedgerEntryRepository socRepo;
    private final SocPiiSanitiser sanitiser;

    @Inject
    SocComplianceService(LedgerVerificationService verificationService,
                         SocLedgerEntryRepository socRepo,
                         SocPiiSanitiser sanitiser) {
        this.verificationService = verificationService;
        this.socRepo = socRepo;
        this.sanitiser = sanitiser;
    }

    public InclusionProof inclusionProof(UUID entryId, String tenancyId) {
        return verificationService.inclusionProof(entryId, tenancyId);
    }

    public List<SocLedgerEntry> incidentTimeline(UUID incidentId, String tenancyId) {
        List<SocLedgerEntry> entries = socRepo.findByIncidentId(incidentId, tenancyId);
        List<SocLedgerEntry> sanitised = new ArrayList<>(entries.size());
        for (SocLedgerEntry entry : entries) {
            SocLedgerEntry copy = new SocLedgerEntry();
            copy.id = entry.id;
            copy.incidentId = entry.incidentId;
            copy.subjectId = entry.subjectId;
            copy.stepType = entry.stepType;
            copy.sequenceNumber = entry.sequenceNumber;
            copy.entryType = entry.entryType;
            copy.actorId = entry.actorId;
            copy.actorRole = entry.actorRole;
            copy.actorType = entry.actorType;
            copy.occurredAt = entry.occurredAt;
            copy.causedByEntryId = entry.causedByEntryId;
            copy.tenancyId = entry.tenancyId;
            copy.metadata = sanitiser.sanitise(entry.metadata);
            sanitised.add(copy);
        }
        return sanitised;
    }

    public DoraResponseTimeReport doraReport(Instant from, Instant to, String tenancyId) {
        List<SocLedgerEntry> entries = socRepo.findByTimeRange(from, to, tenancyId);
        Map<UUID, List<SocLedgerEntry>> byIncident = entries.stream()
                .collect(Collectors.groupingBy(e -> e.incidentId));

        Map<String, List<Duration>> resolutionTimesByPriority = new HashMap<>();
        Map<String, List<Duration>> triageTimesByPriority = new HashMap<>();
        Map<String, List<Duration>> containmentTimesByPriority = new HashMap<>();

        for (var incidentEntries : byIncident.values()) {
            SocLedgerEntry triage = findByType(incidentEntries, SocStepType.ALERT_TRIAGE);
            SocLedgerEntry resolved = findByType(incidentEntries, SocStepType.INCIDENT_RESOLVED);
            SocLedgerEntry containment = findByType(incidentEntries, SocStepType.CONTAINMENT_DECISION);
            if (triage == null) continue;

            String priority = extractPriority(triage.metadata);

            if (resolved != null) {
                Duration total = Duration.between(triage.occurredAt, resolved.occurredAt);
                resolutionTimesByPriority.computeIfAbsent(priority, k -> new ArrayList<>()).add(total);
            }
            triageTimesByPriority.computeIfAbsent(priority, k -> new ArrayList<>()).add(Duration.ZERO);
            if (containment != null) {
                Duration toContainment = Duration.between(triage.occurredAt, containment.occurredAt);
                containmentTimesByPriority.computeIfAbsent(priority, k -> new ArrayList<>()).add(toContainment);
            }
        }

        Map<String, PriorityStats> byPriority = new HashMap<>();
        for (String priority : resolutionTimesByPriority.keySet()) {
            List<Duration> resolutionTimes = resolutionTimesByPriority.getOrDefault(priority, List.of());
            List<Duration> containmentTimes = containmentTimesByPriority.getOrDefault(priority, List.of());
            Duration slaWindow = SLA_WINDOWS.getOrDefault(priority, Duration.ofHours(24));
            long compliant = resolutionTimes.stream().filter(d -> d.compareTo(slaWindow) <= 0).count();
            double slaPercent = resolutionTimes.isEmpty() ? 0.0 : (double) compliant / resolutionTimes.size();

            byPriority.put(priority, new PriorityStats(
                resolutionTimes.size(),
                Duration.ZERO,
                containmentTimes.isEmpty() ? Duration.ZERO : avg(containmentTimes),
                resolutionTimes.isEmpty() ? Duration.ZERO : avg(resolutionTimes),
                slaPercent
            ));
        }

        return new DoraResponseTimeReport(from, to, byIncident.size(), byPriority);
    }

    private static SocLedgerEntry findByType(List<SocLedgerEntry> entries, SocStepType type) {
        return entries.stream().filter(e -> e.stepType == type).findFirst().orElse(null);
    }

    private static String extractPriority(String metadata) {
        if (metadata == null) return "UNKNOWN";
        int idx = metadata.indexOf("\"assignedSeverity\":\"");
        if (idx < 0) return "UNKNOWN";
        int start = idx + "\"assignedSeverity\":\"".length();
        int end = metadata.indexOf("\"", start);
        return end > start ? metadata.substring(start, end) : "UNKNOWN";
    }

    private static Duration avg(List<Duration> durations) {
        long totalMillis = durations.stream().mapToLong(Duration::toMillis).sum();
        return Duration.ofMillis(totalMillis / durations.size());
    }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocComplianceServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Implement SocComplianceResource**

```java
package io.casehub.soc.rest;

import io.casehub.ledger.runtime.service.model.InclusionProof;
import io.casehub.platform.api.identity.CurrentPrincipal;
import io.casehub.soc.domain.DoraResponseTimeReport;
import io.casehub.soc.engine.compliance.SocComplianceService;
import io.casehub.soc.engine.compliance.SocLedgerEntry;
import jakarta.annotation.security.RolesAllowed;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;
import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.PathParam;
import jakarta.ws.rs.QueryParam;
import java.time.Instant;
import java.util.List;
import java.util.UUID;

@Path("/api/soc/compliance")
@ApplicationScoped
@RolesAllowed("soc-compliance-viewer")
public class SocComplianceResource {

    @Inject SocComplianceService service;
    @Inject CurrentPrincipal currentPrincipal;

    @GET @Path("/proof/{entryId}")
    public InclusionProof getProof(@PathParam("entryId") UUID entryId) {
        return service.inclusionProof(entryId, currentPrincipal.tenancyId());
    }

    @GET @Path("/timeline/{incidentId}")
    public List<SocLedgerEntry> getTimeline(@PathParam("incidentId") UUID incidentId) {
        return service.incidentTimeline(incidentId, currentPrincipal.tenancyId());
    }

    @GET @Path("/dora")
    public DoraResponseTimeReport getDoraReport(
            @QueryParam("from") Instant from, @QueryParam("to") Instant to) {
        return service.doraReport(from, to, currentPrincipal.tenancyId());
    }
}
```

- [ ] **Step 6: Run full app build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app -am`
Expected: PASS

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/compliance/SocComplianceService.java app/src/main/java/io/casehub/soc/rest/SocComplianceResource.java app/src/test/java/io/casehub/soc/engine/compliance/SocComplianceServiceTest.java
git commit -m "feat(#24): add SocComplianceService and SocComplianceResource

DORA response time reporting with per-priority aggregation and SLA
compliance. Inclusion proof and PII-sanitised timeline endpoints.
Endpoints secured with @RolesAllowed, tenant-scoped.

Refs #24"
```

---

### Task 7: SocIncidentLedgerObserver — CDI observer for intermediate steps

**Files:**
- Create: `app/src/main/java/io/casehub/soc/engine/compliance/SocIncidentLedgerObserver.java`
- Test: `app/src/test/java/io/casehub/soc/engine/compliance/SocIncidentLedgerObserverTest.java`

**Interfaces:**
- Consumes: `SocLedgerEntryWriter` (Task 5), `SocIncidentStatusChangedEvent` (from #23), `SocIncidentStatus`
- Produces: `SocLedgerEntry` records for ALERT_TRIAGE and INCIDENT_PROMOTED step types

- [ ] **Step 1: Read SocIncidentStatusChangedEvent to understand the CDI event payload**

Use `ide_find_class` for `SocIncidentStatusChangedEvent` and `ide_file_structure` to see its fields. Verify what data it carries (incidentId/caseId, new status, old status, actorId, tenancyId).

- [ ] **Step 2: Write failing test**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.ledger.api.model.LedgerEntry;
import io.casehub.platform.api.identity.ActorType;
import io.casehub.soc.domain.SocIncidentStatus;
import io.casehub.soc.domain.SocIncidentStatusChangedEvent;
import io.casehub.soc.domain.SocStepType;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Clock;
import java.time.Instant;
import java.time.ZoneOffset;
import java.util.UUID;
import static org.assertj.core.api.Assertions.assertThat;

class SocIncidentLedgerObserverTest {

    private SocLedgerEntryWriterTest.CapturingLedgerRepo repo;
    private SocIncidentLedgerObserver observer;

    @BeforeEach
    void setUp() {
        repo = new SocLedgerEntryWriterTest.CapturingLedgerRepo();
        var writer = new SocLedgerEntryWriter(repo,
                Clock.fixed(Instant.parse("2026-08-11T10:00:00Z"), ZoneOffset.UTC));
        observer = new SocIncidentLedgerObserver(writer);
    }

    @Test
    void triagingStatus_writesAlertTriageEntry() {
        UUID caseId = UUID.randomUUID();
        // Adapt constructor based on actual SocIncidentStatusChangedEvent fields
        var event = new SocIncidentStatusChangedEvent(caseId, "tenant-1",
                SocIncidentStatus.DETECTED, SocIncidentStatus.TRIAGING,
                "system:soc-triage", "{\"alertSeverity\":\"CRITICAL\",\"assignedSeverity\":\"HIGH\",\"triageAgentId\":\"agent-1\"}");

        observer.onStatusChanged(event);

        assertThat(repo.lastSaved).isNotNull();
        SocLedgerEntry entry = (SocLedgerEntry) repo.lastSaved;
        assertThat(entry.stepType).isEqualTo(SocStepType.ALERT_TRIAGE);
        assertThat(entry.incidentId).isEqualTo(caseId);
    }

    @Test
    void investigatingStatus_writesIncidentPromotedEntry() {
        UUID caseId = UUID.randomUUID();
        var event = new SocIncidentStatusChangedEvent(caseId, "tenant-1",
                SocIncidentStatus.TRIAGING, SocIncidentStatus.INVESTIGATING,
                "system:soc-triage", "{\"promotionReason\":\"confirmed-threat\"}");

        observer.onStatusChanged(event);

        assertThat(repo.lastSaved).isNotNull();
        SocLedgerEntry entry = (SocLedgerEntry) repo.lastSaved;
        assertThat(entry.stepType).isEqualTo(SocStepType.INCIDENT_PROMOTED);
    }

    @Test
    void irrelevantTransition_skips() {
        UUID caseId = UUID.randomUUID();
        var event = new SocIncidentStatusChangedEvent(caseId, "tenant-1",
                SocIncidentStatus.CONTAINING, SocIncidentStatus.ERADICATED,
                "system:soc", "{}");

        observer.onStatusChanged(event);
        assertThat(repo.lastSaved).isNull();
    }
}
```

**Note:** Adapt the `SocIncidentStatusChangedEvent` constructor and field access based on the actual record definition discovered in Step 1. The test structure above is illustrative — the implementer must verify the event shape.

- [ ] **Step 3: Implement SocIncidentLedgerObserver**

```java
package io.casehub.soc.engine.compliance;

import io.casehub.platform.api.identity.ActorType;
import io.casehub.soc.domain.SocIncidentStatus;
import io.casehub.soc.domain.SocIncidentStatusChangedEvent;
import io.casehub.soc.domain.SocStepType;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.ObservesAsync;
import jakarta.inject.Inject;

@ApplicationScoped
public class SocIncidentLedgerObserver {

    private final SocLedgerEntryWriter writer;

    @Inject
    SocIncidentLedgerObserver(SocLedgerEntryWriter writer) {
        this.writer = writer;
    }

    void onStatusChanged(@ObservesAsync SocIncidentStatusChangedEvent event) {
        SocStepType stepType = mapStatusToStepType(event);
        if (stepType == null) return;

        writer.write(event.caseId(), stepType,
                event.actorId(), "incident-status-change", ActorType.SYSTEM,
                event.metadata(), event.tenancyId(), null);
    }

    private SocStepType mapStatusToStepType(SocIncidentStatusChangedEvent event) {
        // Adapt field access based on actual SocIncidentStatusChangedEvent shape
        if (event.newStatus() == SocIncidentStatus.TRIAGING) return SocStepType.ALERT_TRIAGE;
        if (event.newStatus() == SocIncidentStatus.INVESTIGATING) return SocStepType.INCIDENT_PROMOTED;
        return null;
    }
}
```

**Note:** The exact `SocIncidentStatusChangedEvent` API (field names, accessor methods) must be verified against the actual record from #23. Adapt accordingly.

- [ ] **Step 4: Run test**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=SocIncidentLedgerObserverTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add app/src/main/java/io/casehub/soc/engine/compliance/SocIncidentLedgerObserver.java app/src/test/java/io/casehub/soc/engine/compliance/SocIncidentLedgerObserverTest.java
git commit -m "feat(#24): add SocIncidentLedgerObserver — CDI observer for intermediate steps

Observes SocIncidentStatusChangedEvent for ALERT_TRIAGE and
INCIDENT_PROMOTED ledger entries. Required for DORA report time
delta computation.

Refs #24"
```

---

### Task 8: Full build verification and spec sync

**Files:**
- No new files
- Verify: full project build passes

**Interfaces:**
- Consumes: all prior tasks
- Produces: green build

- [ ] **Step 1: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
Expected: PASS — all tests green

- [ ] **Step 2: Verify all new classes compile and load**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS

- [ ] **Step 3: Verify diagnostics**

Use `ide_diagnostics` on new files to check for compilation errors or warnings.

- [ ] **Step 4: Commit if any fixes were needed**

Only commit if Step 3 revealed issues that needed fixing.
