# JOURNAL — casehub-examples slot 196

## 2026-09-24

### Session: #87 classloader visibility + .m2 rebuild

**Completed:**
- #88 (created & closed): sync wacky-manor with upstream API changes (CbrCaseMemoryStore→CbrRecordStore, StrategyStore evidence methods, StrategyLearningOrchestrator constructor)
- #87 root cause identified and fixed: stale `casehub-platform-agent-claude` JAR in slot-local Maven repo (Sep 15 snapshot missing ClaudeVertexBackendFactory added Sep 22)
- `BackendFactoryDiscoveryTest` added — verifies ClaudeVertexBackendFactory is CDI-discoverable
- Full .m2 rebuild: nuked slot-local repo, symlinked to `~/.m2/repository`, installed platform/engine/blocks/neocortex/eidos/ledger from source repos

**Key decisions:**
- Symlinked `.m2` → `~/.m2/repository` instead of maintaining a separate slot-local repo. Simpler but means slot writes pollute host repo.
- Updated `casehub-ledger-memory` → `casehub-ledger-persistence-memory` (artifact ID renamed upstream)
- Added `quarkus.hibernate-orm.packages=io.casehub.eidos` to restrict JPA scanning
- Extended `quarkus.arc.exclude-types` for JPA stores, eval beans

**Known issues from .m2 rebuild:**
- `quarkus.hibernate-orm.packages` restriction breaks ledger named queries (HHH90003001: Error in named query: LedgerEntry.findSequenceStats) — Quarkus @QuarkusTest fails to start for some tests
- The eidos eval beans are excluded (`io.casehub.eidos.eval.**`) which prevents eval tests from running
- SocialCognitionIntegrationTest not yet verified against eidos personality profile fix
- qhorus repo at HEAD is out of sync with ledger HEAD (compilation failure in QhorusLedgerEntryRepository)
- Engine repo at HEAD is out of sync with ledger HEAD (compilation failure in WorkerDecisionEventCapture)
