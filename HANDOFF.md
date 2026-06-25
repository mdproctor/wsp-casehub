# HANDOFF — casehub

**Date:** 2026-06-25
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**Triage + docs batch: #251 epic closed, #309/#308/#307 docs shipped, local SNAPSHOT rebuild, eviction rethink**

- **#251** (auth retrofit epic): all four harnesses confirmed done (devtown#90, openclaw#41, life#40, clinical#88). Epic closed.
- **#309, #308, #307** (docs batch): iot-api→ops/iot dep row, named MCP server convention, qhorus deep-dive audit/CommitmentContext sync.
- **Local SNAPSHOT rebuild**: full chain parent→engine installed to `.m2` — devtown's reported DEEP_MERGE/SubjectSequenceStats breaks were stale JARs, not API changes.
- **Idle eviction**: launchd daemon unloaded. Script rewritten for on-demand use (`evict-idle-claude.sh 3h`). Stop hook still timestamps every interaction.

## Immediate Next Step

Switch to **casehub-engine session** → engine#543 (Worker primitive migration, 60+ files). Critical path for Worker Foundation Extraction.

## Cross-Module: Worker Foundation Extraction

*Unchanged — `git show HEAD~1:HANDOFF.md`*

## What's Left

- `qhorus#294` — bug: QhorusCloudEventAdapter wrong timestamp (affects Drools CEP) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| engine#543 | Worker primitive migration | L | High | Critical path; do in casehub-engine session |
| #293 | Formalise channel taxonomy | S | Low | No blockers |
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon |
