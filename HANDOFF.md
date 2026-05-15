# Handoff — ContextDiffStrategy CDI Refactor
2026-05-15

## What changed this session

**engine#258 and #261 complete and closed.**

`ContextDiffStrategy` selection refactored from inconsistent CDI annotations to a config-driven `@Produces @DefaultBean` producer. Key insight: `@DefaultBean` is for consumer-replaceable SPIs; the three diff strategies are engine-internal choices, not consumer extension points.

New config: `casehub.engine.diff-strategy=none|top-level|json-patch` (default `none`).

`ContextDiffStrategyProducer` owns instantiation — strategy classes are plain POJOs. `@DefaultBean` goes on the `@Produces` method, not the producer class (garden entry `GE-20260515-fd3156`).

Protocol `PP-20260514-engine-spi-noops-defaultbean` updated with a "two patterns" note distinguishing consumer-replaceable SPIs from engine-internal strategy selection. CLAUDE.md and `docs/repos/casehub-engine.md` synced.

E2E test (`ContextDiffNoneStrategyTest`) uses `@QuarkusTestProfile` to override config and assert no `contextChanges` in EventLog. Podman socket required: `DOCKER_HOST=unix:///run/user/501/podman/podman.sock`.

## Open issues

- `engine#255` — HumanTaskTarget template mode wire-up (inject `WorkItemTemplateService`, call `findById` + `instantiate`)
- `engine#252` — extract `SubCaseCompletionListener` logic into `SubCaseCompletionService`
- `engine#254` — Java 21 platform migration
- `engine#253` — assess quarkus-hibernate-reactive-panache compile-scope dep
- **Devtown** — Epic 3: PR review CasePlanModel (queued multiple sessions)

## Key references

- Blog: `blog/2026-05-15-mdp01-config-over-cdi.md`
- Garden: `GE-20260515-fd3156` (@DefaultBean method placement), `GE-20260515-cd1653` (Podman socket), `GE-20260515-99cf39` (config-driven producer technique)
- Protocol: `parent/docs/protocols/casehub/engine-spi-noops-defaultbean.md`
- Commits: `30fbc53`, `4177037`, `3846527` in `casehub-engine`
