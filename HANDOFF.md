# Handoff — @DefaultBean SPI No-Op Migration
2026-05-14

## What changed this session

**engine#257 complete and closed.**

Nine SPI no-op/empty default beans migrated from bare `@ApplicationScoped` or `@Alternative @ApplicationScoped` to `@DefaultBean @ApplicationScoped` (`io.quarkus.arc.DefaultBean`). Consumer repos (Claudony, devtown) were hitting CDI ambiguity errors when the engine was indexed alongside their `@QuarkusTest` classpath.

**Key gotcha:** `@DefaultBean` is NOT `jakarta.enterprise.inject.DefaultBean` (doesn't exist) — it's `io.quarkus.arc.DefaultBean`. Garden entry `GE-20260514-83ee13` captures this.

Beans fixed (all in `runtime/src/main/java/io/casehub/engine/internal/`):
- `worker/`: NoOpWorkerProvisioner, NoOpCaseChannelProvider, NoOpWorkerStatusListener, EmptyWorkerContextProvider
- `diff/`: NoOpContextDiffStrategy
- `worker/` reactive: NoOpReactiveWorkerProvisioner, NoOpReactiveCaseChannelProvider, NoOpReactiveWorkerStatusListener, EmptyReactiveWorkerContextProvider

**Protocol updated:** `PP-20260514-engine-spi-noops-defaultbean` — all nine beans now listed, correct import noted.

**CLAUDE.md + `casehub-engine.md` deep-dive updated.** "To add a new operational SPI" instruction now includes `@DefaultBean` requirement.

**All artifacts installed to `~/.m2`.** 506 tests pass.

## Follow-up issues

- `engine#255` — HumanTaskTarget template mode wire-up (inject `WorkItemTemplateService`, call `findById` + `instantiate`) — carried from previous session
- `engine#258` — `JsonPatchContextDiffStrategy` still `@Alternative`, inconsistent with no-op now being `@DefaultBean`
- `engine#254` — Java 21 platform migration (coordinate with other repos)
- **Devtown** — Epic 3: PR review CasePlanModel (queued since before this session)

## Key references

- Commits: `6f0e27f`, `6aaaffc`, `555ccd6`, `4c24fd7`, `ded0307` in `casehub-engine`
- Protocol: `parent/docs/protocols/engine-spi-noops-defaultbean.md`
- Blog: `blog/2026-05-14-mdp02-nine-defaults-wrong-package.md`
- Garden: `GE-20260514-83ee13` — @DefaultBean package gotcha
