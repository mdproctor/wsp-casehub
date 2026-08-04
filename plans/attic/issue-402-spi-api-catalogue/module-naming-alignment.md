# Module and Package Naming Alignment Plan

**Status:** Parked — resume after git history cleanup  
**Scope:** All casehubio repos  
**Prerequisite:** Git history squash/cleanup complete on affected repos

---

## The Standard

Follows Quarkus extension convention:

| Dimension | Rule |
|-----------|------|
| groupId | `io.casehub` for all repos |
| Root pom artifactId | `casehub-{repo}-parent` (multi-module) |
| Submodule artifactId | `casehub-{repo}-{function}` — fully qualified |
| Folder name | Short descriptive name — **no repo prefix needed**, the repo provides the context |
| Java package root | `io.casehub.{repo}` |

**Role module folder names** (always short, every extension has them):
`api`, `runtime`, `deployment`, `testing`, `examples`, `integration-tests`

**Capability module folder names** (short, no prefix):
`queues`, `ai`, `ledger`, `notifications`, `blackboard`, `persistence-hibernate`, etc.

The artifactId carries the full qualified name (`casehub-work-queues`). The folder doesn't need to.

---

## Already Correct

**ledger** — `api`, `runtime`, `deployment`, `examples` ✓  
**qhorus** — `api`, `runtime`, `deployment`, `testing`, `examples` ✓ (plus stale `casehub-assisteddev` dir to remove)

---

## Deviations

### work — folders have redundant `casehub-work-` prefix

| Current folder | Should be |
|----------------|-----------|
| `casehub-work-api` | `api` |
| `casehub-work-core` | `core` |
| `casehub-work-ai` | `ai` |
| `casehub-work-ledger` | `ledger` |
| `casehub-work-notifications` | `notifications` |
| `casehub-work-queues` | `queues` |
| `casehub-work-queues-dashboard` | `queues-dashboard` |
| `casehub-work-queues-examples` | `queues-examples` |
| `casehub-work-postgres-broadcaster` | `postgres-broadcaster` |
| `casehub-work-queues-postgres-broadcaster` | `queues-postgres-broadcaster` |
| `casehub-work-persistence-mongodb` | `persistence-mongodb` |
| `casehub-work-reports` | `reports` |
| `casehub-work-examples` | `examples` |
| `casehub-work-flow-examples` | `flow-examples` |
| `casehub-work-issue-tracker` | `issue-tracker` |
| `work-flow` | `flow` |

### engine — folders have inconsistent `casehub-` prefix, plus `engine` folder is ambiguous

| Current folder | Should be |
|----------------|-----------|
| `casehub-blackboard` | `blackboard` |
| `casehub-engine-common` | `common` |
| `casehub-ledger` | `ledger` *(note: currently clashes visually with the ledger repo)* |
| `casehub-persistence-memory` | `persistence-memory` |
| `casehub-persistence-hibernate` | `persistence-hibernate` |
| `casehub-resilience` | `resilience` |
| `casehub-testing` | `testing` |
| `casehub-work-adapter` | `work-adapter` |
| `engine` | `runtime` *(ambiguous — same as repo name)* |

Short names that are already correct: `api`, `codegen`, `schema`, `scheduler-quartz`

### connectors — folders have redundant `connectors-` prefix

| Current folder | Should be |
|----------------|-----------|
| `connectors-core` | `core` |
| `connectors-email` | `email` |

### claudony — folders have redundant `claudony-` prefix

| Current folder | Should be |
|----------------|-----------|
| `claudony-core` | `core` |
| `claudony-casehub` | `casehub` |
| `claudony-app` | `app` |

### devtown — folders have redundant `devtown-` prefix (easy — no published artifacts)

| Current folder | Should be |
|----------------|-----------|
| `devtown-domain` | `domain` |
| `devtown-review` | `review` |
| `devtown-queue` | `queue` |
| `devtown-github` | `github` |
| `devtown-app` | `app` |

Root pom artifactId: `casehub-devtown` → `casehub-devtown-parent`

### qhorus — stale directory

Remove `casehub-assisteddev/` (leftover from before devtown rename).

---

## The One Bigger Change — Claudony groupId and packages

Claudony started as `remotecc` pre-casehubio, became `claudony`, and was never migrated. No design rationale exists for `dev.claudony`.

| Dimension | Current | Should be |
|-----------|---------|-----------|
| groupId | `dev.claudony` | `io.casehub` |
| Root artifactId | `claudony-parent` | `casehub-claudony-parent` |
| Java package | `dev.claudony.*` | `io.casehub.claudony.*` |

**Fix for packages:** IntelliJ Refactor → Move Package on `dev.claudony` → `io.casehub.claudony`. Handles all imports automatically. **Do not use Find+Replace.**

---

## Execution

All folder renames are `git mv` + pom.xml `<module>` entry update. No Java changes needed except claudony.

**Order:**
1. **devtown** — no published artifacts, lowest risk; fix as part of Epic 1 scaffold
2. **claudony** — package rename in IntelliJ first, then folder renames; dedicated session
3. **work** — many renames but mechanical; batch in one session
4. **engine** — same
5. **connectors** — two renames, trivial

Ledger and qhorus need no folder changes.
