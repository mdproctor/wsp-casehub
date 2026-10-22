# Protocol Audit — Earned Inclusion

**Started:** 2026-10-05
**Last updated:** 2026-10-10
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

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/artifact-rename-propagation.md | Rename propagation across consumers (FLAG: casehub-scoped) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/cdi-classpath-presence-requires-module-separation.md | Module separation for @DefaultBean visibility |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/cdi-decorator-unique-priority.md | Unique @Priority values on CDI decorators |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/cdi-observational-decorator-failure-isolation.md | Observer exception isolation in decorators |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/committed-git-hooks.md | Git hooks committed to repo, not local-only |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/cross-org-repo-format-in-ci.md | Full org/repo format in cross-org CI |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/entity-not-null-java-default-matches-sql-default.md | Java defaults match SQL defaults for NOT NULL |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/filesystem-watching-library.md | Use directory-watcher, not WatchService (CSV candidate) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/flyway-extension-migration-registration.md | Extension migration registration (consolidation candidate) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/flyway-repo-scoped-migration-path.md | Repo-scoped migration path (consolidation candidate) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/library-jar-annotation-only-deps.md | Annotation-only deps in library JARs |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/no-jpa-entities-across-requires-new.md | No JPA entity passing across REQUIRES_NEW |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/opaque-cross-module-identifiers.md | Cross-module IDs are opaque strings |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/persistence-backend-cdi-priority.md | Three-tier CDI priority for backends |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-extension-flyway-locations-explicit.md | Explicit Flyway locations in extensions |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-extension-unremovable-consumer-beans.md | Unremovable beans for consumer injection |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-test-database.md | Test database configuration rules |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-void-buildstep-produce-anchor.md | Void BuildStep needs @Produce anchor |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/reactive-messaging-ack-chain-whenComplete.md | Ack chain uses whenComplete not thenApply |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/sanitize-caller-controlled-headers-before-logging.md | Sanitize headers before logging (security) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/scheduled-task-top-level-exception-guard.md | Top-level catch in scheduled tasks |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/spi-signature-change-all-impls-same-commit.md | SPI change + all impls in same commit |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/startup-bean-config-self-contained.md | @Startup beans self-contained config |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/testtransaction-requires-new-unique-names.md | Unique names for REQUIRES_NEW test transactions |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/virtual-threads-default.md | Virtual threads as default (ADR-0005) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/volatile-swap-atomic-single-reference.md | Atomic swap for single-reference fields |

## IN List — CaseHub Protocols (146)

### Engine + Eidos (9)

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-api-scope-rule.md | Depend on engine-api not engine for SPI-only modules |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-case-start-three-phase-pattern.md | Three-phase startCase prevents Agroal pool deadlock |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-runtime-cdi-assembly.md | Engine runtime CDI requires Jandex/JQ/persistence deps |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-spi-action-risk-classifier-qualifier.md | Risk classifier qualifier prevents silent skip |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-spi-noops-defaultbean.md | Engine no-op SPIs must use @DefaultBean |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-worker-event-observer-async.md | @ObservesAsync required for async engine events |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-spi-agent-scope-requires-tenancy-id.md | Eidos SPI methods require tenancyId for data isolation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-vocabulary-metadata-required.md | VocabularyTerm enums require metadata annotation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-vocabulary-registry-register-postconstruct-only.md | VocabularyRegistry.register() only in @PostConstruct |

### Ledger (8)

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-enricher-priority-mandate.md | Enrichers require @Priority — misordering silently corrupts validation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-entry-non-merkle-use-entity-manager.md | Non-Merkle entries use EntityManager.persist not repository.save |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-hash-chain-disabled-in-h2-tests.md | Disable hash chain in H2 tests — H2 lacks row-level locking |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-sequence-cross-subtype-query.md | Sequence query must span all JOINED subtypes |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-spi-propagation.md | LedgerEntryRepository changes must propagate to all impls |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-subclass-extension.md | Ledger subclass rules — JOINED, V1004+, domain-agnostic hash |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-tokenise-for-query-optional-contract.md | tokeniseForQuery Optional contract prevents empty results |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-write-api-transaction-demarcation.md | Write API transaction demarcation prevents async data races |

### Qhorus (8)

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-actor-type-mapping.md | ActorType mapping prevents AI agents classified as HUMAN |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-channel-dual-identity.md | Channels have dual identity: UUID + name |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-consumer-integration-pattern.md | Use QhorusDashboardService for dashboard views |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-dispatch-exception-sanitization.md | Sanitize exceptions before ledger commit (security) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-flyway-consumer-versioning.md | Consumer migrations start at V2000 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-human-governance-channel-types.md | Governance acts require COMMAND not EVENT |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-inbound-normaliser-channel-scope.md | InboundNormaliser must scope to relevant channels |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-per-entity-governance-channels.md | Governance channels named after governed entity |

### Platform + Harness (11)

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/harness-ledger-writer.md | Dedicated writer bean for multi-service ledger writes |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/harness-rest-resource-blocking-applicationscoped.md | REST resources: @Blocking @ApplicationScoped with JDBC |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/harness-tenantid-explicit-parameter.md | Explicit tenantId param for scheduler/async paths |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/harness-transactional-service-layer-only.md | @Transactional on service methods only, not REST |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/harness-workitem-template-test-seeding.md | Seed WorkItemTemplates via @BeforeEach in tests |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-api-scope.md | platform-api scope — cross-cutting concepts only |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-cache-domain-event-bridge.md | Cache invalidation via domain events, not reverse dep |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-cdi-index-dependency.md | Platform CDI index dependency for test discovery |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-mock-bean-exclusion.md | Exclude platform mocks when persistence-memory present |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-spi-contract.md | Platform SPI implementation guide |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-testing-fixed-principal-setup.md | FixedCurrentPrincipal displaces mock in tests |

### SPI + CDI (8)

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/cdi-async-event-tenancyid.md | Async CDI events must carry tenancyId |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/cdi-decorator-agentprovider-cross-cutting.md | @Decorator for cross-cutting AgentProvider concerns |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/cdi-registry-validate-names-at-startup.md | CDI registries validate names at startup |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-deletion-default-throws.md | SPI deletion methods default to throw |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-evolution-default-methods.md | New SPI methods use default to avoid breaking impls |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-placement-caseinstance-goes-in-common.md | CaseInstance SPIs go in common/spi not api/spi |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-property-keys-cross-module-only.md | Property key constants only for cross-module keys |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-test-scope-default-bean-noop.md | Test-scope @DefaultBean no-op for runtime SPIs |

### Work + Channel (19)

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/plan-item-delegated-state.md | PlanItem uses DELEGATED not RUNNING for external actors |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/plan-item-handler-error-path.md | Handler error path must fault PlanItem to prevent hang |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/plan-item-store-blocking-transactional-call-site.md | PlanItemStore writes require blocking transactional context |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/work-adapter-inputmapping-payload-contract.md | InputMapping payload contract for outbound handlers |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/work-adapter-memory-plan-item-store.md | MemoryPlanItemStore required in test alternatives |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/work-adapter-plan-item-running-ordering.md | markDelegated must precede validation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/work-event-type-enum-coverage.md | WorkEventType enum must cover all event types |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/workitem-template-constraints-snapshot-at-instantiation.md | Template constraints snapshot at instantiation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/workload-provider-stub-required-in-tests.md | WorkloadProvider stub required in harness tests |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/case-channel-message-signal.md | Case-channel message signal for Qhorus→engine bridge |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/channel-backend-idempotent-registration.md | Channel backend idempotent registration |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/channel-type-constraint-record-constructor.md | Channel type constraint in record constructor |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/channel-type-policy-invariant.md | Channel type policy invariant — hard vs advisory |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/fleet-channel-notify-no-qhorus-dispatch.md | Fleet notify must not dispatch via Qhorus (loop) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/fleet-key-role-separation.md | Fleet key vs user role separation (security) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/inbound-connector-id-is-type-not-account.md | Connector ID is type, not account |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/inbound-connector-type-separation.md | Inbound vs outbound connector type separation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/message-dispatch-builder-validation.md | Message dispatch builder validation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/message-service-dispatch-enforcement-gate.md | Message service dispatch enforcement gate |

### Cross-cutting + Architecture (83)

| Path | Rule |
|------|------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/connector-service-caller-api.md | Inject ConnectorService, not Instance\<Connector\> |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/openclaw-delivery-always-200.md | OpenClaw delivery always returns 200 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ai-agent-provider-cdi-priority.md | AI agent provider CDI priority for dual impls |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/alternative-extension-patterns.md | Three @Alternative patterns decision tree |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/auth-retrofit-readiness.md | Auth retrofit readiness — no auth in domain/service |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/auto-channel-key-sanitisation.md | Auto channel key sanitisation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/bridge-module-spi-placement.md | Bridge module SPI goes in bridge, not api |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/case-definition-layers.md | Three-layer case definition architecture |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/casehub-dependency-tier-order.md | Dependency tier order — platform-api is universal |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/casehub-engine-flow-module-isolation.md | Engine-flow depends only on engine-common |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/casehub-platform-dependency-scope.md | Platform dependency scope — test vs runtime |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/casehub-work-hibernate-packages.md | Work Hibernate scan requires runtime.model + runtime.filter |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/casehub-work-illegal-state-exception.md | IllegalStateException maps to 409 in work |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/casememorystore-adapter-asserttenant-contract.md | CaseMemoryStore tenant assertion contract (security) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/chatmodel-adapter-unsupported-format-throws.md | ChatModel adapter throws on unsupported format |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ci-dispatch-covers-direct-consumers.md | CI dispatch must cover all direct consumers |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/configmapping-prefix-ownership.md | ConfigMapping prefix ownership — all keys declared |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/consumer-spi-placement.md | Consumer-facing SPI placement — api/spi vs runtime |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/coordinator-no-transactional-multi-datasource.md | No @Transactional at coordinator spanning datasources |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/cross-backend-aggregation-service.md | Cross-backend aggregation isolates failures |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/cross-foundation-bridge-module-placement.md | Cross-foundation bridge module placement |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/cross-repo-optional-dep-table-registration.md | Optional dep table registration for visibility |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/current-principal-boolean-delegates-to-actor-type.md | CurrentPrincipal booleans delegate to ActorType |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/caller-owned-resource-wrapper-no-close.md | Caller-owned resource wrapper must not close |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/claudony-module-boundary-no-app-to-casehub.md | No claudony-app → casehub dependency |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/domain-event-not-workitem-lifecycle-for-triggers.md | Observe domain events, not WorkItemLifecycleEvent |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/domain-supplement-pattern.md | Domain supplement table, not wrapper entity |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/dual-trail-audit-pattern.md | Operational trail vs compliance ledger distinction |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/dual-framework-module-structure.md | Module structure for dual-framework extraction |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/expression-engine-create-paired-with-supports.md | ExpressionEngine.create paired with supports |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/expression-engine-type-invariant.md | ExpressionEngine type invariant — three-way identity |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/failure-cascade-pattern.md | Four-tier agent failure handling reference |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/flyway-consumer-numbering.md | App migrations start at V100+ (work owns V1–V99) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/flyway-ledger-migration-locations.md | Ledger migration locations prevent table-not-found |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/flyway-version-range-allocation.md | Exclusive Flyway thousand-block version ranges |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/hexagonal-application-service-placement.md | Use-case orchestration in app/, api/ stays pure |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/hitl-runtime-assembly.md | HITL runtime assembly for humanTask bindings |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/jackson-externalized-serialization.md | Externalized Jackson serialization — no domain annotations |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/descriptor-handler-pattern.md | Descriptor→handler pattern prevents scattered switches |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/library-jars-require-jandex.md | Library JARs with CDI beans require Jandex index |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/jq-evaluation-canonical.md | Canonical JQ evaluator prevents divergent impls |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/mcp-tool-exception-catch-all.md | MCP tools catch all exceptions — no stack trace leakage |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/library-mcp-named-server.md | Library MCP uses @McpServer("name") not default |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/lifecycle-enum-classification-on-enum.md | Lifecycle classification lives on the enum |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/memory-erase-by-id-completeness.md | Memory erasure must be complete (GDPR) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/memory-storeall-transactional-contract.md | CaseMemoryStore.storeAll transactional contract |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/memory-privileged-ops-no-async-bypass.md | Privileged memory ops must not bypass via async |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/multi-datasource-ledger-work-qhorus.md | Multi-datasource setup for ledger+work+qhorus |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/no-conditional-tenancy-filtering.md | Tenancy filtering always unconditional |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/no-workarounds-fix-the-design.md | Fix the design — no workarounds or shims |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/noop-registry-must-not-fire-cdi-events.md | No-op registry must not fire CDI events |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/normative-channel-layout-single-source.md | Normative channel layout — single source of truth |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/optional-platform-spi-instance-injection.md | Optional platform SPI uses Instance injection |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/orm-xml-tier-separation-api-purity.md | ORM XML tier separation — no JPA in api/ |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/otel-traceid-capture-before-fire-async.md | Capture OTel traceId before fireAsync |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/panel-qualified-jq-expressions.md | Panel-qualified JQ expressions in YAML |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/per-binding-credential-reference.md | Per-binding credential reference, not raw tokens |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/publish-workflow-repository-dispatch.md | Publish workflow requires repository dispatch |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/oidc-harness-wiring-checklist.md | OIDC harness wiring checklist |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/oversight-action-gate-dedicated-hub.md | Oversight action gate uses dedicated hub |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/persistence-hibernate-tenant-aware-repository.md | Hibernate repos must extend TenantAwareRepository |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/new-peer-repo-checklist.md | New peer repo checklist |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/quarkus-maven-plugin-required.md | Quarkus Maven plugin required for CI |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/quarkus-test-spi-override-injectmock.md | Use @InjectMock not @TestProfile+@Alternative |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/rbac-classifier-context-guard.md | RBAC classifier context guard for async paths |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/renderer-cache-key-includes-format.md | Renderer cache key must include format |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/repo-hook-requirements.md | Repo hook requirements for work tracking |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/rest-domain-validation-two-mapper-pattern.md | REST domain validation — two ExceptionMapper pattern |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/scim2-agent-identity-lookup.md | SCIM2 agent identity lookup |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/secondary-write-isolation-after-terminal-commit.md | Secondary writes isolated after terminal commit |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/sse-endpoint-no-virtual-thread.md | SSE endpoints must not use virtual threads |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/static-credentials-config-property-not-preferences.md | Static credentials in config, not Preferences |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/store-owned-ttl-vs-spi-ttl.md | Store owns TTL, not SPI callers |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/tenancy-repository-pattern.md | Bind tenancyId inside data access, not call sites |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/tenancyid-server-side-only.md | tenancyId is server-side only — never in client APIs |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/tutorial-layer-cdi-displacement.md | Tutorial layer CDI displacement ordering |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/typed-preference-keys.md | Typed preference keys, not string keys |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ui-components-pages-blocks-first.md | UI components authored in pages/blocks-ui first |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/webhook-signature-constant-time.md | Webhook signatures use constant-time comparison |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/yaml-humantask-binding-type.md | YAML humanTask uses binding type, not capability |

---

## VERIFY List (36) — needs code check or scope decision

### Universal (8)

| Path | Issue |
|------|-------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/flyway-migration-rules.md | Outdated naming ("quarkus-work" not "casehub-work"), check refs |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/module-tier-structure.md | Core rule critical but mixes universal + casehub content — split? |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/optional-module-pattern.md | Valid rule, outdated groupId examples (io.quarkiverse.work) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/maven-coordinate-standard.md | Entirely casehub-specific — wrong scope, move to casehub |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-optional-extension-dep.md | May be obsolete — gates blocking vs reactive, ADR-0005 retired reactive |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-junit-not-junit5.md | Migration notice — check if platform#19 is complete |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-test-security-http-only.md | Sound but narrow — check if Quarkus now warns |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/spi-adapter-module-placement.md | Design principle, not failure prevention — borderline |

### CaseHub (28)

| Path | Issue |
|------|-------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-cdi-event-await-chain.md | Check if still applies after virtual threads (ADR-0005) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-investigation-test-drain.md | May be narrow to aml — verify other harnesses need drain |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-library-alternative-subclass-defaultbean.md | Scope:repo — verify breadth beyond CaseLedgerEntryRepository |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-vocab-uri-for-axis-not-inlined.md | Guidance severity — verify if inlining caused actual failures |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-algorithm-transparent-signing.md | Scope:repo — only casehub-ledger signing code |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-identity-cache-rotation-invalidation.md | Scope:repo — only casehub-ledger caches |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-test-profile-datasource.md | Scope:repo — specific to QuarkusTestProfile + ledger PU |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-entity-mapper-pure-transformer.md | Scope:repo — check if pattern applies beyond qhorus |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-module-progression.md | Guidance vs failure prevention — verify violations break things |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-case-id-parameter.md | All listed SPIs already comply — verify still prevents new violations |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-event-tenancyid-component-order.md | Convention — verify if violations cause failures or just inconsistency |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/harness-trust-policy-source-split.md | Applies only to devtown TrustRoutingPolicyProvider |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/worker-function-execution-model.md | 200-line protocol reads as guide section, not protocol |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/fleet-client-filter-registration.md | Sound but very narrow (claudony-app only) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/builtin-strategy-registration.md | Narrow (casehub-work only) — verify frequency of new strategies |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/casehub-engine-inputdata-map-convention.md | References open migration issue engine#302 — verify completed |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/claude-agent-provider-vs-tmux-session-choice.md | Verify if tmux pattern still active after ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/case-started-payload-is-panel-document.md | Very narrow — only test code constructing CASE_STARTED events |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/delegation-platform-semantic.md | Narrow to eidos vocabulary producers — verify breadth |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/gateway-backend-idempotent-registration.md | Real failure but narrow — verify applies outside claudony |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/epic-close-code-merge-required.md | Methodology — verify if work-end skill already covers this |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/gateway-backend-registration-ordering.md | Latent hazard — verify any backend does real init in open() |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/jta-tsr-status-active-gate.md | Scope:repo — only qhorus MessageObserverDispatcher |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/journal-migration-number-sync.md | Depends on JOURNAL.md + epic workflow still being used |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/journal-section-anchor.md | Same — depends on JOURNAL.md + epic close still active |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/llm-pass-structural-fallback.md | Only references eidos rendering SPIs — verify broader use |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/local-spi-placeholder-contract-identity.md | Placeholder pattern may be obsolete — verify open cases |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-pg-devservices-test-profile.md | Reactive tier retired — verify if named PG profile still needed |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/stream-discover-default-tenant.md | Narrow (platform-streams-*) — verify modules still exist |

---

## OUT List (69 + 16 RETIRED = 85)

### Retired — ADR-0005 Virtual Threads (16)

| Path | Status |
|------|--------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-sync-async-parity.md | RETIRED — superseded |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/claudony-reactive-spi-variants.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-test-entity-setup-named-pu.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/quarkus-test-stub-reactive-cdi.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-rendered-prompt-cache-canonical-spi.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-spi-bridge-default-bean.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-harness-gate-matches-infrastructure.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-render-pipeline-threading.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-mcp-tool-resolve-channel-blocking.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/reactive-service-build-gating.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/spi-reactive-blocking-io.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/harness-jdbc-only-reactive-suppression.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-reactive-spi-shim.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-reactive-gating.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/reactive-vs-blocking-selection.md | RETIRED — ADR-0005 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/reactive-blocking-tier-separation.md | RETIRED — ADR-0005 |

### Not failure prevention — methodology/process/guidance (25)

| Path | Why out |
|------|---------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/cross-repo-coordination-issues.md | Process guidance, not failure prevention |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/java-optional-usage.md | Style; caught by IDE linters |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/jax-rs-provider-cdi-scope.md | Style consistency, protocol says "not wrong" |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/layer-log.md | Methodology + casehub-specific, wrong location |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/maven-module-scoping.md | Convenience (slow, not broken) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/maven-submodule-folder-naming.md | Cosmetic convention |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/varargs-type-capture.md | Design taste, advisory only |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/vertical-slice-planning.md | Methodology — belongs in building-apps guide |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/platform-ownership-check.md | LLM workflow prompt, not technical rule |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/arc42stories-foundation-tier-layer-taxonomy.md | Documentation methodology |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/arc42stories-source-breadth.md | Documentation methodology |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/claude-md-size-discipline.md | Documentation housekeeping |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/cross-repo-issues-in-parent.md | Process guidance |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/epic-closed-md-deletion-date.md | Branch cleanup process |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/external-api-surface-in-deep-dive.md | Documentation process |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/external-content-modernised-blackboard.md | Branding/terminology |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/issue-scale-complexity-labels.md | Process/methodology |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/no-negative-peer-project-references.md | Marketing/communications |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/peer-session-cross-repo-commit-discipline.md | Process, already in CLAUDE.md |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/protocol-refs-use-double-dotdot.md | Meta — protocol about protocols |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/sweep-blocked-item-process.md | Process/methodology |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/workspace-git-isolation.md | Methodology |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/trust-maturity-model.md | Architecture doc, not protocol |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/subcase-coordination-strategy.md | Design guidance |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/store-owned-retention-mechanism.md | Design principle |

### Too narrow — single repo/call-site (should be repo-level or deleted) (28)

| Path | Why out |
|------|---------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/event-log-left-fold-projection.md | Duplicate of casehub/ version |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/qdrant-client-library.md | CSV candidate — technology recommendation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quartz-ram-store-configuration.md | Casehub-specific, not universal |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/spi-default-method-contract-test.md | Single call-site (devtown) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-axis-exact-match-exhaustive-switch.md | Compiler already enforces this |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-dispositionaxis-mapping-exhaustive-switch.md | Compiler already enforces this |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-enrichment-constants-in-pipeline.md | Single class (EidosRenderPipeline) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-poc-gate-before-jpa-infrastructure.md | Development methodology, not failure prevention |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-validator-constant-per-field.md | Naming readability, not failure |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eidos-vocabulary-metadata-via-registry.md | Single class (EidosRenderPipeline) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/engine-ledger-migration-if-not-exists.md | Single repo (engine-ledger) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/ledger-repository-entitymanager-qualifier.md | Redundant with ledger-subclass-extension |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-message-observer-scope.md | Resolved by qhorus#167 |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-event-content-null.md | Implementation detail, covered by governance-channel-types |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/qhorus-service-store-seam.md | Single repo (qhorus) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/work-adapter-test-subcase-group-repository.md | Single test config |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/channel-create-requires-init-channel.md | Documents correct behavior, no failure to prevent |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/inbound-connector-numeric-metadata-always-present.md | Convenience, not failure |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/claudony-channel-backend-registration-path.md | Single call site (claudony) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/claudony-sse-timing-preference-key.md | Single repo (claudony) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/openclaw-hook-typescript-only.md | Single repo (openclaw) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/agent-routing-strategy-guard-before-emiton.md | Single implementation |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/debate-entry-type-encoding-standard.md | Single repo (drafthouse) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/delivery-audit-raw-output.md | Single repo (openclaw) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/deployment-spec-tenant-scoped.md | Single repo (ops) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/disc-vocab-is-disposition.md | Single repo (eidos) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/endpoint-registered-no-op-must-not-fire.md | Single call site |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eval-dimension-format-only-gate.md | Single repo (eidos eval) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/eval-variant-pair-single-axis-isolation.md | Already enforced by code |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/event-log-left-fold-projection.md | Design pattern, not failure prevention + duplicate |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/flyway-data-migration-naming-dependency.md | Single repo (qhorus) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/format-specific-enrichment-schema-isolation.md | Performance, not failure (eidos) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/hitl-test-context-pre-seeding.md | Single repo (devtown) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/inmemory-store-aggregate-no-scan-delegation.md | Single repo (qhorus-testing) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/jpa-mapper-positional-constructor-field-completeness.md | Single method |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/plugin-hooks-call-rest-not-mcp.md | Single repo (openclaw) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/provisioner-spi-provision-result.md | Compiler enforces via type signature |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/python-component-in-maven-repo.md | Single repo (openclaw) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/quarkus-test-lifecycle-entity-setup.md | Single repo (clinical) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/render-format-structure-naming.md | Single enum (eidos-api) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/runtime-test-cdi-exclude-ledger-capture-beans.md | Single test profile (engine) |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/subcase-tenancyid-inherits-parent.md | Single handler (engine) — keep as repo-level |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/tmux-test-await-not-sleep.md | Single repo (claudony) |

---

## Relocation Flags

### Should move from garden to repo-level (IN but narrow scope)

| Path | Target repo |
|------|-------------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/blackboard-registry-call-order.md | engine |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/blackboard-registry-get-blocking-required.md | engine |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/plan-item-delegated-state.md | engine |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/plan-item-store-blocking-transactional-call-site.md | engine |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/work-adapter-inputmapping-payload-contract.md | engine |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/work-adapter-plan-item-running-ordering.md | engine |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/fleet-channel-notify-no-qhorus-dispatch.md | claudony |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/fleet-key-role-separation.md | claudony |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/claudony-module-boundary-no-app-to-casehub.md | claudony |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/openclaw-delivery-always-200.md | openclaw |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/agent-descriptor-compact-constructor-validation.md | eidos |

### Should move from universal to casehub-scoped

| Path | Reason |
|------|--------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/artifact-rename-propagation.md | Entirely casehub-specific |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/maven-coordinate-standard.md | VERIFY — io.casehub groupId |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quartz-ram-store-configuration.md | OUT — casehub-specific |

### Should move to building-apps guide

| Path | Reason |
|------|--------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/casehub/trust-maturity-model.md | Architecture doc, not protocol |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/vertical-slice-planning.md | Methodology |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/layer-log.md | Methodology |

### Dependency preferences → CSV

| Path | Reason |
|------|--------|
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/filesystem-watching-library.md | Library preference |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/qdrant-client-library.md | Library preference |
| /Users/mdproctor/claude/casehub/garden/docs/protocols/universal/quarkus-junit-not-junit5.md | Migration, may be done |

---

## Next Steps

1. **Delete** 16 retired + duplicates + resolved protocols
2. **Relocate** ~11 IN protocols from garden to repo-level
3. **Extract** 3 dependency preferences to CSV
4. **Resolve** 36 VERIFY protocols (code-level checks)
5. **Rebuild indexes** with consumer/contributor terminology
6. **Triage** hortora strays (10) and repo-level protocols (~100)
