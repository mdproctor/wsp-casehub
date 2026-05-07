# Module and Package Naming Alignment Plan

**Status:** Parked — resume after git history cleanup  
**Scope:** All casehubio repos  
**Prerequisite:** Git history squash/cleanup complete on affected repos (ledger is mid-squash-branch)

---

## The Standard

| Dimension | Rule |
|-----------|------|
| groupId | `io.casehub` for all repos |
| Root pom artifactId | `casehub-{repo}-parent` (multi-module) |
| Submodule artifactId | `casehub-{repo}-{function}` |
| Runtime artifact | `casehub-{repo}` (no function suffix) |
| Folder name | Must match artifactId exactly |
| Java package root | `io.casehub.{repo}` |

---

## Deviations

### 1. Folder names don't match artifactId — generic names used

These repos use short generic folder names (`api`, `runtime`, `deployment`, `testing`, `examples`) that hide the full artifact identity. Folder should equal artifactId.

| Repo | Folder → ArtifactId (current) | Folder should be |
|------|-------------------------------|-----------------|
| ledger | `api` → `casehub-ledger-api` | `casehub-ledger-api` |
| ledger | `runtime` → `casehub-ledger` | `casehub-ledger` |
| ledger | `deployment` → `casehub-ledger-deployment` | `casehub-ledger-deployment` |
| qhorus | `api` → `casehub-qhorus-api` | `casehub-qhorus-api` |
| qhorus | `runtime` → `casehub-qhorus` | `casehub-qhorus` |
| qhorus | `deployment` → `casehub-qhorus-deployment` | `casehub-qhorus-deployment` |
| qhorus | `testing` → `casehub-qhorus-testing` | `casehub-qhorus-testing` |
| qhorus | `examples` → `casehub-qhorus-examples` | `casehub-qhorus-examples` |
| work | `runtime` → `casehub-work` | `casehub-work` |
| work | `deployment` → `casehub-work-deployment` | `casehub-work-deployment` |
| work | `testing` → `casehub-work-testing` | `casehub-work-testing` |
| work | `work-flow` → `casehub-work-flow` | `casehub-work-flow` |
| work | `integration-tests` → `casehub-work-integration-tests` | `casehub-work-integration-tests` |
| engine | `api` → `casehub-engine-api` | `casehub-engine-api` |
| engine | `codegen` → `casehub-engine-codegen` | `casehub-engine-codegen` |
| engine | `schema` → `casehub-engine-schema` | `casehub-engine-schema` |
| engine | `engine` → `casehub-engine` | `casehub-engine` |
| engine | `scheduler-quartz` → `casehub-engine-scheduler-quartz` | `casehub-engine-scheduler-quartz` |

**Fix:** `git mv` folder, update `<module>` in parent pom.xml. No Java changes needed.

---

### 2. Engine folders have `casehub-` prefix but missing `-engine-`

These engine submodules use `casehub-{thing}` instead of `casehub-engine-{thing}`, creating ambiguity with top-level repos.

| Folder (current) | ArtifactId | Folder should be |
|-----------------|-----------|-----------------|
| `casehub-persistence-memory` | `casehub-engine-persistence-memory` | `casehub-engine-persistence-memory` |
| `casehub-persistence-hibernate` | `casehub-engine-persistence-hibernate` | `casehub-engine-persistence-hibernate` |
| `casehub-blackboard` | `casehub-engine-blackboard` | `casehub-engine-blackboard` |
| `casehub-resilience` | `casehub-engine-resilience` | `casehub-engine-resilience` |
| `casehub-ledger` | `casehub-engine-ledger` | `casehub-engine-ledger` |
| `casehub-testing` | `casehub-engine-testing` | `casehub-engine-testing` |
| `casehub-work-adapter` | `casehub-engine-work-adapter` | `casehub-engine-work-adapter` |

**Fix:** `git mv` + pom.xml `<module>` update. Note: `casehub-ledger` folder name currently clashes visually with the `casehub-ledger` repo — high priority.

---

### 3. Connectors folders missing `casehub-` prefix

| Folder (current) | ArtifactId | Folder should be |
|-----------------|-----------|-----------------|
| `connectors-core` | `casehub-connectors` | `casehub-connectors` |
| `connectors-email` | `casehub-connectors-email` | `casehub-connectors-email` |

**Fix:** `git mv` + pom.xml update.

---

### 4. Devtown submodules missing `casehub-devtown-` prefix

| Folder (current) | ArtifactId (current) | Should be |
|-----------------|---------------------|-----------|
| `devtown-domain` | `devtown-domain` | `casehub-devtown-domain` |
| `devtown-review` | `devtown-review` | `casehub-devtown-review` |
| `devtown-queue` | `devtown-queue` | `casehub-devtown-queue` |
| `devtown-github` | `devtown-github` | `casehub-devtown-github` |
| `devtown-app` | `devtown-app` | `casehub-devtown-app` |

Root pom: `casehub-devtown` → `casehub-devtown-parent`

**Fix:** `git mv` + pom.xml update. Low risk — no published artifacts yet.

---

### 5. Claudony groupId, artifactId, and packages (biggest change)

Claudony started as `remotecc` (pre-casehubio), became `claudony`, and was never migrated to platform conventions when joining casehubio. There is no design rationale for `dev.claudony` — it is historical accident.

| Dimension | Current | Should be |
|-----------|---------|-----------|
| groupId | `dev.claudony` | `io.casehub` |
| Root artifactId | `claudony-parent` | `casehub-claudony-parent` |
| `claudony-core` folder + artifact | same | `casehub-claudony-core` |
| `claudony-casehub` folder + artifact | same | `casehub-claudony-casehub` |
| `claudony-app` folder + artifact | same | `casehub-claudony-app` |
| Java package | `dev.claudony.*` | `io.casehub.claudony.*` |

**Fix:**
1. Package rename `dev.claudony` → `io.casehub.claudony` **in IntelliJ** (Refactor → Move package) — handles all imports automatically
2. `git mv` folders
3. pom.xml groupId/artifactId edits
4. Published artifact groupId changes — any consumers of `dev.claudony:*` must update

---

## Execution Order

1. **Claudony** — highest impact, most changes, do in dedicated session with IntelliJ open
2. **Engine** — folder renames only, no Java changes; do after git history cleanup
3. **Ledger, qhorus, work, connectors** — folder renames only; batch in one session
4. **Devtown** — low risk (no published artifacts); do alongside or after scaffold epic

## Notes

- IntelliJ Refactor → Move handles all Java package/import changes atomically. Do not use Find+Replace for the Java side.
- Folder renames (`git mv`) + pom.xml `<module>` updates are the rest of the work — bash + Edit tools.
- The claudony package rename changes the published artifact coordinates. Any downstream that depends on `dev.claudony:claudony-*` will need updating — currently only `casehub-engine` (the engine-ledger module) might reference it, check before cutting the release.
