# Layer 5: Compliance & Audit Evidence — Design Spec

**Date:** 2026-08-11
**Issue:** casehubio/soc#24
**Branch:** issue-21-trust-cbr-compliance
**Status:** Draft

---

## Overview

Create a tamper-evident audit trail for SOC incident investigations. Each investigation step produces a `SocLedgerEntry` — a JPA subclass of `JpaLedgerEntry` — whose domain fields are Merkle-hashed for compliance evidence. Expose JAX-RS endpoints for Merkle inclusion proofs, incident ledger timelines, and DORA response time reports. Add PII sanitisation for decision context before compliance reporting.

**Deliberate divergences from issue #24 acceptance criteria:**
- **Single entity vs 6 subclasses:** One `SocLedgerEntry` with a `stepType` enum replaces 6 separate `JpaLedgerEntry` subclasses. Follows the `CaseLedgerEntry` pattern (one entity, string-typed event differentiation). Step-specific data in `metadata` JSON (already Merkle-hashed via `canonicalBytes()`). Application-level validation enforces compliance-critical fields per step type.
- **`DecisionContextSanitiser` SPI:** Does not exist in the platform. Building a local SOC sanitiser with regex-based PII detection. Person name detection deferred.
- **Flyway migration:** Single `V5000__soc_ledger_entry.sql` instead of 6 separate table migrations.

---

## Components

### 1. SocStepType (api/)

Enum in `io.casehub.soc.domain` defining investigation step types for ledger entries.

```java
public enum SocStepType {
    ALERT_TRIAGE,
    INCIDENT_PROMOTED,
    INVESTIGATION_STEP,
    CONTAINMENT_DECISION,
    CONTAINMENT_EXECUTED,
    INCIDENT_RESOLVED
}
```

### 2. SocLedgerEntry (app/)

JPA entity extending `JpaLedgerEntry`. Single join table, differentiated by `stepType`.

```java
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
}
```

- `incidentId` equals `subjectId` — domain-explicit FK for readability, set once at creation. Enforced by `@PrePersist` that throws `IllegalStateException` if `incidentId != subjectId`.
- `stepType` — which investigation phase this entry records
- `metadata` (inherited) — step-specific JSON payload, already included in `canonicalBytes()`

**Subject identity:** `subjectId` (and `incidentId`) is always the CaseInstance UUID. For `ALERT_TRIAGE` entries, the case already exists (alert ingestion creates the case before triage begins). There is no pre-case triage step — the `SiemAlertGanglion` fires detection, which creates the case, which then triggers the triage binding.

**Compliance-critical field validation:** `SocLedgerEntryObserver` validates required metadata fields per step type before saving. For `CONTAINMENT_DECISION`, the metadata must include `approverId` and `riskClassification` — the observer throws `IllegalStateException` if missing, preventing an incomplete compliance record from entering the Merkle chain.

### 3. Flyway Migration (app/)

`app/src/main/resources/db/soc/migration/V5000__soc_ledger_entry.sql`:

```sql
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

**Configuration changes:**
- `application.properties`: add `classpath:db/soc/migration` to `quarkus.flyway.qhorus.locations`
- Test `application.properties` (to be created): add `casehub.ledger.hash-chain.enabled=false`, add SOC ledger package to `quarkus.hibernate-orm.qhorus.packages`

### 4a. SocIncidentLedgerObserver (app/)

`@ApplicationScoped` CDI observer in `io.casehub.soc.engine.compliance`. Handles intermediate investigation steps via `@ObservesAsync`.

**Event sources for intermediate steps:**
- `SocIncidentStatusChangedEvent` (from #23) → `ALERT_TRIAGE`, `INCIDENT_PROMOTED`
- `WorkerDecisionEntry` CDI events → `INVESTIGATION_STEP`
- WorkItem completion events → `CONTAINMENT_DECISION`, `CONTAINMENT_EXECUTED`

The specific event types depend on what the engine fires during binding execution — verified during implementation.

### 4b. SocResolutionLedgerObserver (app/)

`@ApplicationScoped` service in `io.casehub.soc.engine.compliance` implementing `CaseOutcomeObserver`. Handles `INCIDENT_RESOLVED` entries at case completion. Follows the existing SPI pattern (`SocAttestationService`, `SocCbrRetainService`).

### 4c. SocLedgerEntryWriter (app/)

Shared write helper in `io.casehub.soc.engine.compliance`. Called by both observers. Owns sequence number assignment, metadata validation, and `LedgerEntryRepository.save()` invocation.

**Write protocol (per GE-20260511-b6f903, GE-20260612-17c161):**
1. Create `SocLedgerEntry` instance
2. Set caller-managed fields: `subjectId` = `incidentId`, `entryType` = `LedgerEntryType.EVENT`, `actorId`, `actorRole`, `occurredAt`
3. Set `sequenceNumber` from `LedgerEntryRepository.findLatestBySubjectId()` + 1
4. Set `metadata` JSON with step-specific payload
5. Call `LedgerEntryRepository.save(entry, tenancyId)` — handles Merkle hash, signing
6. Never call `em.persist()` directly (blocked by build step)

**Concurrency:** Sequence number assignment uses find-latest + increment, which has a TOCTOU race under concurrent observers for the same incident. Per GE-20260531-d2ed26, concurrent writes to the same `subjectId` risk Merkle frontier constraint violations. Mitigation: the observer runs in `QuarkusTransaction.requiringNew()` with serializable isolation for same-incident writes. In practice, investigation steps are sequential (each binding fires after the previous completes), so true concurrency is rare.

**Validation per step type:**

| stepType | Required metadata fields |
|---|---|
| ALERT_TRIAGE | alertSeverity, assignedSeverity, triageAgentId |
| INCIDENT_PROMOTED | promotionReason |
| INVESTIGATION_STEP | capabilityTag, investigationType |
| CONTAINMENT_DECISION | approverId, riskClassification, containmentAction |
| CONTAINMENT_EXECUTED | executionResult, containmentAction |
| INCIDENT_RESOLVED | resolutionOutcome |

### 5. SocPiiSanitiser (app/)

`@ApplicationScoped` in `io.casehub.soc.engine.compliance`. Local SOC interface — not a platform SPI.

Strips PII from JSON string values using regex:
- **IPv4:** `\b\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}\b` → `[REDACTED-IP]`
- **IPv6:** standard forms including compressed (`::1`), mapped v4 (`::ffff:10.0.0.1`), and link-local with zone IDs → `[REDACTED-IP]`
- **Email:** `\b[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}\b` → `[REDACTED-EMAIL]`

**Not covered in v1:** Person names (requires NER — file follow-on issue). Hostnames with PII patterns (e.g., `john-smiths-laptop`) — low priority, hostname is typically an IOC indicator, not PII in the compliance context.

**Application point:** Called by `SocComplianceService` before returning timeline or report data. Does not modify stored data — sanitisation is presentational (report-time redaction), not destructive. GDPR Art.17 erasure is handled by the platform's `LedgerErasureService` which operates on `actorId` tokenisation.

Preserves: ATT&CK IDs (T####), severity values, action types, timestamps, agent IDs, UUIDs.

### 6. SocLedgerEntryRepository (app/)

`@ApplicationScoped` in `io.casehub.soc.engine.compliance`. Typed queries for `SocLedgerEntry`. Follows the `CaseLedgerEntryRepository` / `MessageLedgerEntryRepository` platform pattern.

**Named queries:**
- `SocLedgerEntry.findByIncidentId` — `SELECT e FROM LedgerEntry e WHERE TYPE(e) = SocLedgerEntry AND e.subjectId = :incidentId AND e.tenancyId = :tenancyId ORDER BY e.sequenceNumber ASC`
- `SocLedgerEntry.findByTimeRange` — `SELECT e FROM LedgerEntry e WHERE TYPE(e) = SocLedgerEntry AND e.occurredAt >= :from AND e.occurredAt <= :to AND e.tenancyId = :tenancyId ORDER BY e.occurredAt ASC`
- `SocLedgerEntry.findByStepType` — `SELECT e FROM LedgerEntry e WHERE TYPE(e) = SocLedgerEntry AND TREAT(e AS SocLedgerEntry).stepType = :stepType AND e.tenancyId = :tenancyId ORDER BY e.occurredAt ASC`

### 7. SocPreferences (api/)

`PreferenceKey` constants class in `io.casehub.soc.domain`. Defines SLA response windows per priority level. Created as part of this issue — does not exist yet.

```java
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

### 8. SocComplianceService (app/)

`@ApplicationScoped` in `io.casehub.soc.engine.compliance`. Owns query logic and report building.

**Methods:**

```java
InclusionProof inclusionProof(UUID entryId, String tenancyId)
List<SocLedgerEntry> incidentTimeline(UUID incidentId, String tenancyId)
DoraResponseTimeReport doraReport(Instant from, Instant to, String tenancyId)
```

- `inclusionProof` — delegates to `LedgerVerificationService.inclusionProof()`
- `incidentTimeline` — queries `SocLedgerEntryRepository.findByIncidentId()`, applies `SocPiiSanitiser` before returning
- `doraReport` — queries `SocLedgerEntryRepository.findByTimeRange()` for SOC entries in the window, groups by incident, computes durations between step types (`ALERT_TRIAGE.occurredAt` to `INCIDENT_RESOLVED.occurredAt` for end-to-end), aggregates by priority (read from triage entry metadata `assignedSeverity`)

### 9. DoraResponseTimeReport (api/)

Record in `io.casehub.soc.domain`:

```java
public record DoraResponseTimeReport(
    Instant reportPeriodStart,
    Instant reportPeriodEnd,
    int totalIncidents,
    Map<String, PriorityStats> byPriority) {}

public record PriorityStats(
    int count,
    Duration avgTimeToTriage,
    Duration avgTimeToContainment,
    Duration avgTimeToResolution,
    double slaCompliancePercent) {}
```

**SLA compliance calculation:** Percentage of incidents where `INCIDENT_RESOLVED.occurredAt - ALERT_TRIAGE.occurredAt` is within the SLA window for that priority level. SLA windows come from `SocPreferences` — a new `PreferenceKey` constants class created as part of this issue. Defaults: P1 (CRITICAL): 15min, P2 (HIGH): 1hr, P3 (MEDIUM): 4hr, P4 (LOW): 24hr.

**Tenancy:** All queries scoped by `tenancyId`. Reports are per-tenant. Cross-tenant aggregation is not supported in v1.

### 10. SocComplianceResource (app/)

JAX-RS resource in `io.casehub.soc.rest`. Thin shell delegating to `SocComplianceService`.

```java
@Path("/api/soc/compliance")
@ApplicationScoped
@RolesAllowed("soc-compliance-viewer")
public class SocComplianceResource {

    @GET @Path("/proof/{entryId}")
    public InclusionProof getProof(@PathParam("entryId") UUID entryId)

    @GET @Path("/timeline/{incidentId}")
    public List<SocLedgerEntry> getTimeline(@PathParam("incidentId") UUID incidentId)

    @GET @Path("/dora")
    public DoraResponseTimeReport getDoraReport(
        @QueryParam("from") Instant from,
        @QueryParam("to") Instant to)
}
```

All endpoints scoped to tenant via `CurrentPrincipal.tenancyId()`.

RBAC: `@RolesAllowed("soc-compliance-viewer")` at class level. Consider splitting to `soc-compliance-admin` for proof verification if needed later.

---

## Data Flow

```
Investigation step completes (binding fires, WorkItem resolves, etc.)
    │
    ├─ Intermediate step → SocIncidentLedgerObserver (@ObservesAsync CDI)
    │                           │
    └─ Case completion  → SocResolutionLedgerObserver (CaseOutcomeObserver SPI)
                                │
                                ▼
                    SocLedgerEntryWriter (shared)
                        │  determines SocStepType from event signal
                        │  builds metadata JSON from event payload
                        │  validates compliance-critical fields per step type
                        │
                        ├─ LedgerEntryRepository.findLatestBySubjectId()
                        │     → sequenceNumber = latest + 1
                        │
                        └─ LedgerEntryRepository.save(socLedgerEntry, tenancyId)
                              → Merkle hash computed, entry persisted

SocComplianceResource (read path)
    │
    ├─ /proof/{entryId} → LedgerVerificationService.inclusionProof()
    │     → InclusionProof with Merkle path + root
    │
    ├─ /timeline/{incidentId} → SocLedgerEntryRepository.findByIncidentId()
    │     → PII sanitised, ordered SocLedgerEntry list
    │
    └─ /dora?from=...&to=... → SocLedgerEntryRepository.findByTimeRange()
          → DoraResponseTimeReport with per-priority stats
```

---

## Error Handling

| Scenario | Behaviour |
|---|---|
| Missing compliance-critical metadata field | `IllegalStateException` — entry not created, event logged at ERROR |
| `LedgerEntryRepository.save()` fails | Log error, continue. Failed entry permanently lost — no retry mechanism |
| Sequence number race (concurrent observers) | `QuarkusTransaction.requiringNew()` with serializable isolation. In practice, investigation steps are sequential |
| Entry not found for proof request | `LedgerVerificationService` throws `IllegalArgumentException` → 404 response |
| No entries for incident timeline | Return empty list |
| DORA report with no data in range | Return report with `totalIncidents = 0`, empty `byPriority` map |
| PII sanitiser failure | Replace the entry's metadata with `[SANITISATION_FAILED]` and log at ERROR. Fail-closed — an incorrect compliance report is worse than an unavailable one |

---

## Testing Strategy

| Level | What | How |
|---|---|---|
| Unit | `SocStepType` enum values | Plain JUnit |
| Unit | `SocLedgerEntry.domainContentBytes()` | Plain JUnit |
| Unit | `SocPiiSanitiser` — IPv4, IPv6, email patterns; preserves ATT&CK IDs, UUIDs | Plain JUnit, parameterized |
| Unit | `DoraResponseTimeReport` construction and SLA calculation | Plain JUnit |
| Unit | `SocComplianceService.doraReport()` aggregation logic | Mock repositories |
| Unit | Compliance-critical field validation per step type | Plain JUnit |
| Integration | Full ledger entry write via `LedgerEntryRepository.save()` | `@QuarkusTest` with in-memory ledger, hash chain disabled |
| Integration | Timeline endpoint returns ordered, PII-sanitised entries | `@QuarkusTest` |
| Integration | DORA endpoint aggregates correctly | `@QuarkusTest` |

**Test configuration:**
- `casehub.ledger.hash-chain.enabled=false` (per PP-20260604-f45c95)
- `quarkus.flyway.qhorus.locations` includes `classpath:db/soc/migration`
- `quarkus.hibernate-orm.qhorus.packages` includes SOC ledger entity package

---

## Files Changed

| File | Change |
|---|---|
| `api/.../domain/SocStepType.java` | New — step type enum |
| `api/.../domain/DoraResponseTimeReport.java` | New — DORA report record |
| `api/.../domain/PriorityStats.java` | New — per-priority stats record |
| `api/.../domain/SocPreferences.java` | New — SLA window PreferenceKey constants |
| `app/.../compliance/SocLedgerEntry.java` | New — JpaLedgerEntry subclass |
| `app/.../compliance/SocIncidentLedgerObserver.java` | New — CDI observer for intermediate steps |
| `app/.../compliance/SocResolutionLedgerObserver.java` | New — CaseOutcomeObserver for resolution |
| `app/.../compliance/SocLedgerEntryWriter.java` | New — shared write helper |
| `app/.../compliance/SocLedgerEntryRepository.java` | New — typed queries |
| `app/.../compliance/SocComplianceService.java` | New — query/aggregation |
| `app/.../compliance/SocPiiSanitiser.java` | New — regex PII redaction |
| `app/.../rest/SocComplianceResource.java` | New — JAX-RS endpoints |
| `app/src/main/resources/db/soc/migration/V5000__soc_ledger_entry.sql` | New — join table |
| `app/src/main/resources/application.properties` | Add `db/soc/migration` to qhorus flyway locations |
| Tests for all new classes | New |
