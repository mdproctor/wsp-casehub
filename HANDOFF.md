# Handoff — 2026-05-21

**Head commit (engine):** a31dd6e — fix(api): validate humanTask conflicting fields and expiresIn in CaseDefinitionYamlMapper (Closes #297)
**Head commit (workspace):** 30b54f5 — docs: add blog entry 2026-05-21 — P1D Was Never Invalid

## What Changed This Session

Closed engine#297 — three validations added to `CaseDefinitionYamlMapper.convertHumanTask`:
- Both `title` + `templateRef` present → throws `IllegalArgumentException`
- Invalid `expiresIn` format (e.g. `"1h"`) → wraps `DateTimeParseException` as IAE with value context
- Zero or negative `expiresIn` → throws `IllegalArgumentException`
Four tests added; `DateTimeParseException` import added. Gotcha: `Duration.parse("P1D")` is valid Java (86400s) — GE-20260521-1e981c in jvm garden.

Also closed engine#304 (GitHub issue only — work was already in codebase).

Merged `epic-io-thread-safety` workspace branch to main (2 commits that never landed). All workspace branches now properly closed.

Published 7 casehub blog backlog to mdproctor.github.io (May 15–21).

## Immediate Next Step

Pick up the next issue from the engine open list — several S/M items are open. engine#314 (CaseContextImpl.evalObjectTemplate nested interpolation) and engine#312 (HumanTaskScheduleHandler skips WorkItem creation) were both opened 2026-05-21 and look like bugs worth tackling.

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#314 | CaseContextImpl.evalObjectTemplate doesn't recurse nested `{...}` — returns String literal | S | Med | Bug opened 2026-05-21 |
| engine#312 | HumanTaskScheduleHandler skips WorkItem creation (PlanningStrategyLoopControl pre-marks RUNNING) | S | Med | Bug opened 2026-05-21 |
| engine#315 | WorkItemLifecycleAdapter: runOnSafeVertxContext times out in complex deployments | S | High | Vertx threading, opened 2026-05-21 |
| engine#301 | Typed CommandContent record to replace raw Map in COMMAND dispatch | S | Low | Design follow-on to #300 |
| engine#303 | SpiWiringIntegrationTest provisioner tests timing-sensitive, fail without Docker | S | Med | Test stability |
| engine#304 | (closed this session) | — | — | Done |
| claudony#122 | Extract correlationId + deadline from COMMAND content | S | Med | engine#300 now closed, unblocked |
| engine#26 | Design: zero-dep platform-api module | L | High | Multi-repo, significant scope |

## Key References

- Blog: `blog/2026-05-21-mdp02-p1d-was-never-invalid.md`
- Garden: GE-20260521-1e981c (Duration.parse("P1D") valid Java)
- Spec: `docs/specs/2026-05-21-yaml-mapper-error-handling-design.md`
