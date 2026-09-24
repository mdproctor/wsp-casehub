# HANDOFF — casehub-examples

## Last Session

Fixed #87 (ClaudeVertexBackendFactory classloader visibility) and #88 (upstream API sync). Rebuilt the slot's `.m2` from scratch — symlinked to host `~/.m2/repository` and installed all platform modules from source.

### What was done

- **#88** (created & closed): wacky-manor compilation fixes for upstream API changes — `CbrCaseMemoryStore` → `CbrRecordStore` (neocortex rename), `StrategyStore` gained evidence methods (blocks#294), `StrategyLearningOrchestrator` constructor dropped `CbrRecordStore` param
- **#87** root cause: slot-local `.m2/` had stale `casehub-platform-agent-claude` JAR from Sep 15 missing `ClaudeVertexBackendFactory` (added Sep 22). Fix: rebuilt `.m2` from source repos, added `BackendFactoryDiscoveryTest` verifying CDI discovery
- Updated `casehub-ledger-memory` → `casehub-ledger-persistence-memory` (artifact ID renamed upstream)
- Added `quarkus.hibernate-orm.packages=io.casehub.eidos` and extended `quarkus.arc.exclude-types` for JPA stores and eval beans

### Previous Session

Completed blocks#294 (blocks-social-jpa) from examples slot. All 8 plan tasks implemented — 3 new Maven modules, StrategyLearningOrchestrator refactored. PR casehubio/blocks#295 created.

## Immediate Next Step

Queue position 0/6 — #87 is done but `.plan` hasn't been advanced yet. Run `work next` to advance to #86 (NeedTierMappingProvider empty map fallback).

## Critical: .m2 State

The slot's `.m2` is now a **symlink** to `~/.m2/repository`, not a separate directory. This means:
- No offline mode (`-o`) — offline rejects locally-installed SNAPSHOTs from `host-m2` remote
- Online builds work but may hit GitHub Packages (mirror config redirects `github,github-casehubio,github-*` to host-m2)
- Slot-level `slot-settings.xml` has the mirror config; per-repo `.mvn/slot-settings.xml` in examples also has it
- `quarkus.hibernate-orm.packages=io.casehub.eidos` restricts JPA scanning — needed because ledger JPA entities reference types not on classpath. Side effect: breaks ledger named queries (`HHH90003001: Error in named query: LedgerEntry.findSequenceStats`), causing some `@QuarkusTest` tests to fail at startup

### repos installed from source to `~/.m2/repository`:
- `platform` (agent-api, agent-claude, agent-router, agent-gate, agent-config, credentials, llm-config, identity + deps)
- `engine` (from slot 204 — full install including runtime)
- `blocks` (blocks + deps)
- `neocortex` (memory-api, memory, memory-core, memory-cbr-inmem, mindmap-*, cognitive-index)
- `eidos` (runtime, core, api, vocab, persistence-memory, eval + deployment)
- `ledger` (full install including deployment, persistence-memory, jpa-common — installed with `-Dmaven.repo.local=~/.m2/repository` to override ledger's own `.mvn/maven.config`)
- `qhorus` (runtime, api, persistence-memory — deployment module from GitHub Packages, not source-built due to qhorus/ledger API mismatch at HEAD)
- `work` (progress-api)

## Known Issues from .m2 Rebuild

1. **Hibernate named query failure** — `quarkus.hibernate-orm.packages=io.casehub.eidos` excludes ledger entities but their named queries still register, causing startup errors in some `@QuarkusTest` tests
2. **Eidos eval excluded** — `io.casehub.eidos.eval.**` in exclude-types prevents eval judge beans from loading; eval tests won't run
3. **User reports eidos now renders personality profiles properly** — SocialCognitionIntegrationTest should be re-verified once Hibernate issue is resolved
4. **Cross-repo API mismatches at HEAD** — qhorus references removed ledger types (`LedgerMerkleFrontierRepository`, `LedgerPersistenceUnit`); engine references removed ledger field (`domainData`)

## Test Suite Note

565 tests, 1 failure (SocialCognitionIntegrationTest — may be fixed by eidos improvement, needs Hibernate fix first), 0 errors. Online builds required (no `-o` with symlinked .m2).

## References

| Artifact | Path |
|----------|------|
| Phase D spec | specs/phase-d-cognitive-activation/2026-09-20-phase-d-cognitive-activation-design.md |
| Phase D plan | plans/2026-09-20-phase-d-cognitive-activation.md |
| Audit report | audits/2026-09-19-cognitive-architecture-audit.md |
