## D1: LedgerEntry subclass strategy

**Choice:** Single `SocLedgerEntry` class with `stepType` enum field, following the `CaseLedgerEntry` pattern
**Alternatives:**
- 6 separate subclasses (AlertTriage, IncidentPromoted, InvestigationStep, ContainmentDecision, ContainmentExecuted, IncidentResolved) — type-safe per-step fields, schema-enforced NOT NULL, but 6 join tables, 6 migrations, 6 entities to maintain
**Rationale:** CaseLedgerEntry proves the single-entity pattern at platform level. SOC entries share the same query axis (by incidentId, ordered by sequence). Pre-release: cheaper to evolve one entity than six. Step-specific data goes in typed nullable columns and the `metadata` JSON field (already Merkle-hashed via `canonicalBytes()`).
**Trade-offs:** Weaker compile-time enforcement on per-step fields; nullable columns instead of NOT NULL per subtype
**Exploration:** quick
**Status:** captured

## D2: SocLedgerEntry join table columns

**Choice:** Two typed columns: `incident_id` (UUID, equals subjectId — domain readability) and `step_type` (VARCHAR enum). All step-specific data goes in `metadata` JSON.
**Alternatives:**
- Additional typed columns for severity, agent_id_ref, confidence — queryable without JSON operators, but duplicates base-class fields or creates sparse columns
- step_type only (no incident_id) — subjectId already carries the value, but loses domain readability
**Rationale:** Mirrors CaseLedgerEntry's `caseId` pattern — domain-explicit FK alongside the generic `subjectId`. `step_type` is the only SOC-specific discriminator needed for queries and `domainContentBytes()`. Step-specific payloads in `metadata` keeps the schema stable as step types evolve.
**Trade-offs:** `incident_id` is a redundant copy of `subjectId` — must be kept in sync (set once at creation, never updated)
**Exploration:** quick
**Status:** captured

## D3: Compliance layer structure

**Choice:** `SocComplianceService` for query/aggregation logic + `SocComplianceResource` as thin JAX-RS shell
**Alternatives:**
- Inline queries in resource — simpler, fewer classes, but DORA aggregation logic untestable without HTTP
**Rationale:** DORA report builds non-trivial aggregation (group by priority, compute durations between step types, SLA compliance percentages). Service layer enables plain JUnit tests of the aggregation logic independent of JAX-RS.
**Trade-offs:** One extra class; service/resource split for what starts as two endpoints
**Exploration:** quick
**Status:** captured

## D4: PII sanitisation approach

**Choice:** Regex-only sanitiser for IPv4/IPv6, email addresses, and hostnames. Person name detection deferred. Local SOC interface (not a platform SPI — `DecisionContextSanitiser` doesn't exist upstream).
**Alternatives:**
- Regex + heuristic name detection (camelCase/firstName-lastName hostname patterns) — catches more PII but unreliable, false positives on legitimate hostnames
- Full NER-based approach — requires ML model dependency, heavyweight for v1
**Rationale:** IPs and emails are reliably regex-matchable and are the highest-value PII targets in SOC decision context. Person name detection without NER is unreliable. File follow-on issue for name detection when platform gets NER capability.
**Trade-offs:** Person names in decision context JSON are not sanitised in v1
**Depends on:** D3 (sanitiser is called by the compliance service layer)
**Exploration:** quick
**Status:** captured

## D5: Ledger entry write trigger

**Choice:** CDI event observers — single `SocLedgerEntryObserver` observes `CaseLifecycleEvent` and maps event signals to `SocStepType` values, writing `SocLedgerEntry` via `LedgerEntryRepository.save()`
**Alternatives:**
- Explicit factory calls from SOC services — each service calls `SocLedgerEntryFactory.record()` at the right point; tighter coupling, pollutes pipeline code
**Rationale:** Follows established SOC CDI observer pattern (SocAttestationService, SocCbrRetainService). Decoupled from case pipeline. Platform already fires CaseLifecycleEvent at every transition with full case file snapshot.
**Trade-offs:** Observer indirection makes the write trigger less visible in the pipeline code; debugging requires knowing the CDI event flow
**Exploration:** quick
**Status:** captured

## D6: Flyway migration placement and versioning

**Choice:** `db/soc/migration/` location, V5000 range. First migration: `V5000__soc_ledger_entry.sql`. Added to qhorus flyway locations in application.properties.
**Alternatives:**
- Put migrations in `db/ledger/migration` — owned by casehub-ledger, SOC shouldn't write there
- Use `db/migration` (default) — would run against the work datasource, not the qhorus/ledger datasource
**Rationale:** Each module owns its own Flyway location and version range (protocol PP-20260508-f6a8e5). V5000 is the next unclaimed block. Qhorus datasource owns the ledger tables, so SOC migrations must be added to `quarkus.flyway.qhorus.locations`.
**Trade-offs:** None significant — standard module isolation pattern
**Depends on:** D1 (single entity means one CREATE TABLE migration)
**Exploration:** quick
**Status:** captured
