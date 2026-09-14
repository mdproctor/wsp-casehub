# HANDOFF — casehub-soc

## Last Session

Fixed #34 (upstream API drift) by excluding broken CDI beans: CaseMemoryObserver (removed MemoryEmitter type), IdentityBeans (unproxyable producer outputs), and ledger identity enrichers. Built #52 CrowdStrike Falcon connector — OAuth2 client, JAX-RS endpoint at `/crowdstrike/containment`, health check, 20 tests (unit + WireMock contract). Integration test re-enablement blocked by qhorus SNAPSHOT drift mid-session.

## Immediate Next Step

Wait for qhorus#438 fix, then re-enable `ConnectorContainmentIntegrationTest`. After that, `work next` advances to #53 (Palo Alto connector).

## Cross-Module

- casehubio/engine#1094 — CaseMemoryObserver references removed MemoryEmitter
- casehubio/parent#475 — IdentityBeans produces unproxyable @ApplicationScoped beans
- casehubio/qhorus#438 — SNAPSHOT CDI deployment errors (blocks integration test re-enablement)

## Garden Entries Consulted

GE-20260418-9b272f, GE-20260427-edbacd, GE-20260421-1192cd

## References

- `specs/issue-34-fix-upstream-api-breaks/2026-09-14-crowdstrike-falcon-connector-design.md`
- `plans/2026-09-14-crowdstrike-falcon-connector.md`
- `blog/2026-09-14-mdp02-dependency-drift-archaeology.md`
