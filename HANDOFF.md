# HANDOFF — casehub

**Date:** 2026-06-25
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**Triage + docs batch + build fixes: #251 epic closed, #309/#308/#307 shipped, build-all clone failures fixed**

- **#251** (auth retrofit epic): all four harnesses done → epic closed.
- **#309, #308, #307** (docs batch): iot-api dep row, named MCP server convention, qhorus deep-dive sync.
- **Local SNAPSHOT rebuild**: full chain parent→engine. Devtown's reported breaks were stale JARs, not API changes.
- **Idle eviction**: launchd daemon unloaded, script rewritten for on-demand use (`evict-idle-claude.sh 3h`).
- **build-all fixes** (post-close): added `worker` → `casehubio/casehub-worker` REPO_OVERRIDE in CI; disabled `quarkus-langchain4j` in modules-local.csv (forked build no longer needed).

## Immediate Next Step

Switch to **casehub-engine session** → engine#543 (Worker primitive migration, 60+ files). Critical path for Worker Foundation Extraction.

## Cross-Module: Worker Foundation Extraction

*Unchanged — `git show HEAD~2:HANDOFF.md`*

## What's Left

- `qhorus#294` — bug: QhorusCloudEventAdapter wrong timestamp (affects Drools CEP) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#543 | Worker primitive migration | L | High | Critical path; do in casehub-engine session |
| #293 | Formalise channel taxonomy | S | Low | No blockers |
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon |
