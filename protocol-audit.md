# Protocol Audit — Earned Inclusion

**Started:** 2026-10-05
**Last updated:** 2026-10-09
**Approach:** Empty "in" list. Every protocol justifies itself or stays out.

## Criteria

A protocol earns inclusion if it answers YES to all three:
1. **Prevents a real failure** — not caught by compiler, tests, or CI
2. **Broad enough** — applies to more than a single call site in a single repo
3. **Still accurate** — references code/patterns that exist today

## Scope Summary

| Location | Total | Reviewed | In | Out | Verify | Retired |
|----------|-------|----------|----|-----|--------|---------|
| Garden universal/ | 48 | 48 | 26 | 12 | 8 | 2 |
| Garden casehub/ | 245 | 245 | 146 | 57 | 28 | 14 |
| Hortora strays | 10 | 0 | — | — | — | — |
| Repo-level | ~100 | 0 | — | — | — | — |
| **Total** | **~403** | **293** | **172** | **69** | **36** | **16** |

**Garden triage complete.** 172 IN (59%), 69 OUT (24%), 36 VERIFY (12%), 16 RETIRED (5%).

## Terminology Decision

- **Consumer** = builds apps on the platform (was "Harness" / "Application")
- **Contributor** = extends the platform itself (was "Foundation" / "Platform")
- Consumer/contributor is an **audience label**, not a file name — works for protocols and metadata (`scope: consumer`), not for guide names (`building-apps.md`)

## Structural Decisions

- **Dependency preferences** → extract to `preferred-dependencies.csv` (greppable, lintable), not protocol files
- **Methodology/process rules** → move to guide sections or skill rules, not protocols
- **Repo-scoped rules** → move from garden to repo `docs/protocols/`

---

## IN List — Universal Protocols (26)

| File | Rule |
|------|------|
| artifact-rename-propagation.md | Rename propagation across consumers (FLAG: casehub-scoped) |
| cdi-classpath-presence-requires-module-separation.md | Module separation for @DefaultBean visibility |
| cdi-decorator-unique-priority.md | Unique @Priority values on CDI decorators |
| cdi-observational-decorator-failure-isolation.md | Observer exception isolation in decorators |
| committed-git-hooks.md | Git hooks committed to repo, not local-only |
| cross-org-repo-format-in-ci.md | Full org/repo format in cross-org CI |
| entity-not-null-java-default-matches-sql-default.md | Java defaults match SQL defaults for NOT NULL |
| filesystem-watching-library.md | Use directory-watcher, not WatchService (CSV candidate) |
| flyway-extension-migration-registration.md | Extension migration registration (consolidation candidate) |
| flyway-repo-scoped-migration-path.md | Repo-scoped migration path (consolidation candidate) |
| library-jar-annotation-only-deps.md | Annotation-only deps in library JARs |
| no-jpa-entities-across-requires-new.md | No JPA entity passing across REQUIRES_NEW |
| opaque-cross-module-identifiers.md | Cross-module IDs are opaque strings |
| persistence-backend-cdi-priority.md | Three-tier CDI priority for backends |
| quarkus-extension-flyway-locations-explicit.md | Explicit Flyway locations in extensions |
| quarkus-extension-unremovable-consumer-beans.md | Unremovable beans for consumer injection |
| quarkus-test-database.md | Test database configuration rules |
| quarkus-void-buildstep-produce-anchor.md | Void BuildStep needs @Produce anchor |
| reactive-messaging-ack-chain-whenComplete.md | Ack chain uses whenComplete not thenApply |
| sanitize-caller-controlled-headers-before-logging.md | Sanitize headers before logging (security) |
| scheduled-task-top-level-exception-guard.md | Top-level catch in scheduled tasks |
| spi-signature-change-all-impls-same-commit.md | SPI change + all impls in same commit |
| startup-bean-config-self-contained.md | @Startup beans self-contained config |
| testtransaction-requires-new-unique-names.md | Unique names for REQUIRES_NEW test transactions |
| virtual-threads-default.md | Virtual threads as default (ADR-0005) |
| volatile-swap-atomic-single-reference.md | Atomic swap for single-reference fields |

## IN List — CaseHub Protocols (146)

### Engine (9)
engine-api-scope-rule, engine-case-start-three-phase-pattern, engine-runtime-cdi-assembly, engine-spi-action-risk-classifier-qualifier, engine-spi-noops-defaultbean, engine-worker-event-observer-async, eidos-spi-agent-scope-requires-tenancy-id, eidos-vocabulary-metadata-required, eidos-vocabulary-registry-register-postconstruct-only

### Ledger (10)
ledger-enricher-priority-mandate, ledger-entry-non-merkle-use-entity-manager, ledger-hash-chain-disabled-in-h2-tests, ledger-sequence-cross-subtype-query, ledger-spi-propagation, ledger-subclass-extension, ledger-tokenise-for-query-optional-contract, ledger-write-api-transaction-demarcation

### Qhorus (10)
qhorus-actor-type-mapping, qhorus-channel-dual-identity, qhorus-consumer-integration-pattern, qhorus-dispatch-exception-sanitization, qhorus-flyway-consumer-versioning, qhorus-human-governance-channel-types, qhorus-inbound-normaliser-channel-scope, qhorus-per-entity-governance-channels

### Platform + Harness (13)
harness-ledger-writer, harness-rest-resource-blocking-applicationscoped, harness-tenantid-explicit-parameter, harness-transactional-service-layer-only, harness-workitem-template-test-seeding, platform-api-scope, platform-cache-domain-event-bridge, platform-cdi-index-dependency, platform-mock-bean-exclusion, platform-spi-contract, platform-testing-fixed-principal-setup

### SPI + CDI (10)
cdi-async-event-tenancyid, cdi-decorator-agentprovider-cross-cutting, cdi-registry-validate-names-at-startup, spi-deletion-default-throws, spi-evolution-default-methods, spi-placement-caseinstance-goes-in-common, spi-property-keys-cross-module-only, spi-test-scope-default-bean-noop

### Work + Channel (19)
plan-item-delegated-state, plan-item-handler-error-path, plan-item-store-blocking-transactional-call-site, work-adapter-inputmapping-payload-contract, work-adapter-memory-plan-item-store, work-adapter-plan-item-running-ordering, work-event-type-enum-coverage, workitem-template-constraints-snapshot-at-instantiation, workload-provider-stub-required-in-tests, case-channel-message-signal, channel-backend-idempotent-registration, channel-type-constraint-record-constructor, channel-type-policy-invariant, fleet-channel-notify-no-qhorus-dispatch, fleet-key-role-separation, inbound-connector-id-is-type-not-account, inbound-connector-type-separation, message-dispatch-builder-validation, message-service-dispatch-enforcement-gate

### Cross-cutting + Architecture (remaining)
connector-service-caller-api, openclaw-delivery-always-200, ai-agent-provider-cdi-priority, alternative-extension-patterns, auth-retrofit-readiness, auto-channel-key-sanitisation, bridge-module-spi-placement, case-definition-layers, casehub-dependency-tier-order, casehub-engine-flow-module-isolation, casehub-platform-dependency-scope, casehub-work-hibernate-packages, casehub-work-illegal-state-exception, casememorystore-adapter-asserttenant-contract, chatmodel-adapter-unsupported-format-throws, ci-dispatch-covers-direct-consumers, configmapping-prefix-ownership, consumer-spi-placement, coordinator-no-transactional-multi-datasource, cross-backend-aggregation-service, cross-foundation-bridge-module-placement, cross-repo-optional-dep-table-registration, current-principal-boolean-delegates-to-actor-type, caller-owned-resource-wrapper-no-close, claudony-module-boundary-no-app-to-casehub, domain-event-not-workitem-lifecycle-for-triggers, domain-supplement-pattern, dual-trail-audit-pattern, dual-framework-module-structure, expression-engine-create-paired-with-supports, expression-engine-type-invariant, failure-cascade-pattern, flyway-consumer-numbering, flyway-ledger-migration-locations, flyway-version-range-allocation, hexagonal-application-service-placement, hitl-runtime-assembly, jackson-externalized-serialization, descriptor-handler-pattern, library-jars-require-jandex, jq-evaluation-canonical, mcp-tool-exception-catch-all, library-mcp-named-server, lifecycle-enum-classification-on-enum, memory-erase-by-id-completeness, memory-storeall-transactional-contract, memory-privileged-ops-no-async-bypass, multi-datasource-ledger-work-qhorus, no-conditional-tenancy-filtering, no-workarounds-fix-the-design, noop-registry-must-not-fire-cdi-events, normative-channel-layout-single-source, optional-platform-spi-instance-injection, orm-xml-tier-separation-api-purity, otel-traceid-capture-before-fire-async, panel-qualified-jq-expressions, per-binding-credential-reference, publish-workflow-repository-dispatch, oidc-harness-wiring-checklist, oversight-action-gate-dedicated-hub, persistence-hibernate-tenant-aware-repository, new-peer-repo-checklist, quarkus-maven-plugin-required, quarkus-test-spi-override-injectmock, rbac-classifier-context-guard, renderer-cache-key-includes-format, repo-hook-requirements, rest-domain-validation-two-mapper-pattern, scim2-agent-identity-lookup, secondary-write-isolation-after-terminal-commit, sse-endpoint-no-virtual-thread, static-credentials-config-property-not-preferences, store-owned-ttl-vs-spi-ttl, tenancy-repository-pattern, tenancyid-server-side-only, tutorial-layer-cdi-displacement, typed-preference-keys, ui-components-pages-blocks-first, webhook-signature-constant-time, yaml-humantask-binding-type

---

## VERIFY List (36) — needs code check or scope decision

### Universal (8)
flyway-migration-rules, module-tier-structure, optional-module-pattern, maven-coordinate-standard, quarkus-optional-extension-dep, quarkus-junit-not-junit5, quarkus-test-security-http-only, spi-adapter-module-placement

### CaseHub (28)
engine-cdi-event-await-chain, engine-investigation-test-drain, engine-library-alternative-subclass-defaultbean, eidos-vocab-uri-for-axis-not-inlined, ledger-algorithm-transparent-signing, ledger-identity-cache-rotation-invalidation, ledger-test-profile-datasource, qhorus-entity-mapper-pure-transformer, platform-module-progression, spi-case-id-parameter, spi-event-tenancyid-component-order, harness-trust-policy-source-split, worker-function-execution-model, fleet-client-filter-registration, builtin-strategy-registration, casehub-engine-inputdata-map-convention, claude-agent-provider-vs-tmux-session-choice, case-started-payload-is-panel-document, delegation-platform-semantic, gateway-backend-idempotent-registration, epic-close-code-merge-required, gateway-backend-registration-ordering, jta-tsr-status-active-gate, journal-migration-number-sync, journal-section-anchor, llm-pass-structural-fallback, local-spi-placeholder-contract-identity, reactive-pg-devservices-test-profile, stream-discover-default-tenant

---

## OUT List (69 + 16 RETIRED = 85)

### Retired — ADR-0005 Virtual Threads (16)
ledger-sync-async-parity, claudony-reactive-spi-variants, reactive-test-entity-setup-named-pu, quarkus-test-stub-reactive-cdi, reactive-rendered-prompt-cache-canonical-spi, reactive-spi-bridge-default-bean, reactive-harness-gate-matches-infrastructure, reactive-render-pipeline-threading, reactive-mcp-tool-resolve-channel-blocking, reactive-service-build-gating, spi-reactive-blocking-io, harness-jdbc-only-reactive-suppression, ledger-reactive-spi-shim, qhorus-reactive-gating, reactive-vs-blocking-selection, reactive-blocking-tier-separation

### Not failure prevention — methodology/process/guidance (18)
cross-repo-coordination-issues, java-optional-usage, jax-rs-provider-cdi-scope, layer-log, maven-module-scoping, maven-submodule-folder-naming, varargs-type-capture, vertical-slice-planning, platform-ownership-check, arc42stories-foundation-tier-layer-taxonomy, arc42stories-source-breadth, claude-md-size-discipline, cross-repo-issues-in-parent, epic-closed-md-deletion-date, external-api-surface-in-deep-dive, external-content-modernised-blackboard, issue-scale-complexity-labels, no-negative-peer-project-references, peer-session-cross-repo-commit-discipline, protocol-refs-use-double-dotdot, sweep-blocked-item-process, workspace-git-isolation, trust-maturity-model, subcase-coordination-strategy, store-owned-retention-mechanism

### Too narrow — single repo/call-site (should be repo-level or deleted) (27)
event-log-left-fold-projection (duplicate), qdrant-client-library (CSV), quartz-ram-store-configuration (casehub-scoped), spi-default-method-contract-test (devtown), eidos-axis-exact-match-exhaustive-switch, eidos-dispositionaxis-mapping-exhaustive-switch, eidos-enrichment-constants-in-pipeline, eidos-poc-gate-before-jpa-infrastructure, eidos-validator-constant-per-field, eidos-vocabulary-metadata-via-registry, engine-ledger-migration-if-not-exists, ledger-repository-entitymanager-qualifier, qhorus-message-observer-scope (resolved), qhorus-event-content-null, qhorus-service-store-seam, work-adapter-test-subcase-group-repository, channel-create-requires-init-channel, inbound-connector-numeric-metadata-always-present, claudony-channel-backend-registration-path, claudony-sse-timing-preference-key, openclaw-hook-typescript-only, agent-routing-strategy-guard-before-emiton, debate-entry-type-encoding-standard, delivery-audit-raw-output, deployment-spec-tenant-scoped, disc-vocab-is-disposition, endpoint-registered-no-op-must-not-fire, eval-dimension-format-only-gate, eval-variant-pair-single-axis-isolation, flyway-data-migration-naming-dependency, format-specific-enrichment-schema-isolation, hitl-test-context-pre-seeding, inmemory-store-aggregate-no-scan-delegation, jpa-mapper-positional-constructor-field-completeness, plugin-hooks-call-rest-not-mcp, provisioner-spi-provision-result, python-component-in-maven-repo, quarkus-test-lifecycle-entity-setup, render-format-structure-naming, runtime-test-cdi-exclude-ledger-capture-beans, subcase-tenancyid-inherits-parent, tmux-test-await-not-sleep

---

## Relocation Flags

### Should move from garden to repo-level (IN but narrow scope)
- blackboard-registry-call-order → engine
- blackboard-registry-get-blocking-required → engine
- plan-item-delegated-state → engine
- plan-item-store-blocking-transactional-call-site → engine
- work-adapter-inputmapping-payload-contract → engine
- work-adapter-plan-item-running-ordering → engine
- fleet-channel-notify-no-qhorus-dispatch → claudony
- fleet-key-role-separation → claudony
- claudony-module-boundary-no-app-to-casehub → claudony
- openclaw-delivery-always-200 → openclaw
- agent-descriptor-compact-constructor-validation → eidos

### Should move from universal to casehub-scoped
- artifact-rename-propagation
- maven-coordinate-standard (VERIFY)
- quartz-ram-store-configuration (OUT)

### Should move to building-apps guide
- trust-maturity-model
- vertical-slice-planning
- layer-log

### Dependency preferences → CSV
- filesystem-watching-library
- qdrant-client-library
- quarkus-junit-not-junit5 (migration, may be done)

---

## Next Steps

1. **Delete** 16 retired + duplicates + resolved protocols
2. **Relocate** ~11 IN protocols from garden to repo-level
3. **Extract** 3 dependency preferences to CSV
4. **Resolve** 36 VERIFY protocols (code-level checks)
5. **Rebuild indexes** with consumer/contributor terminology
6. **Triage** hortora strays (10) and repo-level protocols (~100)
