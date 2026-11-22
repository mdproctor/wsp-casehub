# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-22a:** Completed 7 issues (#964, #1080, #1189, #1190, #1193, #1194, #1196). Queue drained. Blog written. All repos published.

**Queue position:** Drained. #1089 deferred (unbounded cache growth, XS/Low).

## What Was Done — Session 22a

### Issues Landed

| Issue | Repo | Summary |
|-------|------|---------|
| #964 | eidos | YAML round-trip fix + 3 tests for ampdFacets deserialization in DispositionDeserializer |
| #1080 | neocortex | CDI observer for AgentDescriptorUpdated → EffectiveCognitiveDefaults cache invalidation |
| #1189 | neocortex | Unified 3 LLM call paths in extract_structure.py via user_prompt parameter |
| #1190 | neocortex | 5 edge-case tests for EdgeInferencePhase.parseRelationships regex fragility |
| #1193 | neocortex | 3 unit tests for SituationalSnapshotAccessorProducer mapping logic |
| #1194 | neocortex | trustAccessor numeric effect test — βR amplification verification |
| #1196 | neocortex | Standardised 8 orchestrator ConcurrentHashMap key formats to agentId:tenantId |

### Key Finding

Code review surfaced a missing null guard on ampdFacets YAML parsing — `facetNode.get("facet")` would NPE if the YAML object was missing either field. Fixed to match the adjacent situationalProfiles parsing pattern.

### Blog

- `2026-11-22-mdp01-the-gaps-nobody-sees.md` — infrastructure gaps in cognitive architectures: silent deserialization failures, stale caches, inconsistent key formats

### Research

- Updated `era8-cognitive-pipeline-validation-research.md` — added 7 issues to landed table

## Queue — next items

Drained.

### Deferred

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1089 | Unbounded cache growth situational personality | XS | Low |

## Not Landed (pre-existing conflicts from other branches)

- `wsp-casehub-neocortex:issue-1098-bdi-semantic-classifier` — conflicts in blog files
- `wsp-casehub-agency:issue-1098-bdi-semantic-classifier` — conflicts in .queue

These are workspace repos from a different branch, not from this session's work.

## Neocortex Build Note

```bash
git -C neocortex fetch canonical main && git -C neocortex rebase canonical/main
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/pom.xml install -Dmaven.test.skip=true -s neocortex/.mvn/slot-settings.xml -T1C
```
