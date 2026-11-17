# HANDOFF — casehub-examples (slot 196)

## Current State

**Session 2026-11-17h (continued):** Issue house-cleaning audit: 126→117 open issues. Closed 9 (7 completed epics + 2 verified-fixed). Identified 10 stale literature research epics (kept open as roadmap backlog). Reopened #591 (Theory of Mind) — core ToM landed but full BToM scope incomplete. Audited ToM wiring: `MentalModelOrchestrator` (LLM-based BDI) is fully wired end-to-end; `TheoryOfMindEngine` (Bayesian) has zero production callers. Filed #1094–#1096 for ToM quick wins. All 3 repos published to remotes.

**Session 2026-11-17h:** Resolved rebase conflicts and closed #712 + #1055. Rebased neocortex branch onto canonical main (32 commits ahead from slot 211 #1070-1072). Semantic conflict resolution in 3 files / 11 regions — merged EffectiveCognitiveDefaults migration with PersonalityMetadataEnricher PID-5 enrichment. Code review: 0 CRITICAL, 2 WARNING (deferred as #1085, #1086), 6 NOTEs (#1087-#1089). Blog: "The Rebase Nobody Asked For". Squash: 11→10 commits. Close ceremony: all 3 repos landed + published. Slot 196 marked ready for archive.

**All branches landed and published.** No active branch work in this slot.

## What Was Landed

### #712 — Shadow archetypes: situation-dependent personality degradation (neocortex, 10 commits)
Situational personality modulation: elastic band model where personality facets displace under stress and return toward resting values. Config loader, integrator, mode detector (gain-form evidence model), state service (MindMap persistence), tick participant (TERMINAL phase), consolidation phase (SituationalDeformationPhase), journal entries, E2E integration + noise-floor topology tests.

### #1055 — EffectiveCognitiveDefaults (neocortex, included in #712 branch)
Tenant-scoped cognitive defaults with resting/effective split. LearningProfile type-safety wrapper for resting consumers. DescriptorViewSource and PersonalityStateSource SPIs. Consumer migration across cognition, mindmap-intelligence, cognitive-eidos (49 files).

### eidos#234 — Situational profiles (eidos, 5 commits)
AmpdFacetShift, SituationalProfile records. Archetype-family profiles for 12 families × 5 modes. SituationalProfileDeriver for family-default injection. 26 tests.

## Issue Audit Summary

**Closed this session (9):**

| # | Title | Reason |
|---|-------|--------|
| #960 | Derivation pipeline observability | Landed |
| #65 | memory-memori epic | All children done |
| #471 | cognitive workbench epic | 9/9 children done |
| #576 | Cognitive Workbench multi-corpus epic | All children done |
| #595 | Coping strategies epic | 15/15 children done |
| #599 | Narrative identity epic | Child done |
| #789 | Corpus & workspace platform epic | 8/8 children done |
| #1049 | CognitiveEmergenceSampleTest distrust | Verified fixed by #1044 |
| #1078 | CopingStrategyPredictorTest arity | Verified fixed by #1052 |

**Stale literature epics (10, kept open as research roadmap):** #538, #591, #593, #594, #597, #598, #600, #602, #603, #604

## ToM Wiring Findings

- **MentalModelOrchestrator** (LLM BDI): fully wired — signals → LLM inference → prompt section → consolidation
- **TheoryOfMindEngine** (Bayesian): built, tested, zero production callers — the quantitative channel is disconnected
- **ToM → Trust**: not connected — inferred intentions don't affect trust scores

## Issues Filed This Session

| # | Title | Scale | Complexity |
|---|-------|-------|-----------|
| #1085 | Restore appraisal weight enrichment in classification metadata | S | Low |
| #1086 | PersonalityMetadataEnricher orphaned after EffectiveCognitiveDefaults migration | XS | Low |
| #1087 | SituationalStateService: log unrecognised mode names during deserialization | XS | Low |
| #1088 | SituationalStateService.loadOrCreate TOCTOU — use computeIfAbsent | XS | Low |
| #1089 | Unbounded cache growth in situational personality components | XS | Low |
| **#1094** | **Wire TheoryOfMindEngine into CognitionCore tick** | **S** | **Low** |
| **#1095** | **Feed MentalModel BDI into trust evolution** | **S** | **Med** |
| **#1096** | **Surface Bayesian ToM posterior in prompt section** | **XS** | **Low** |

## Queue (next session starts here)

#1094 (Wire TheoryOfMindEngine) is the recommended first pick — biggest impact, all components exist, wiring-only change.

| # | Title | Scale | Complexity | Notes |
|---|-------|-------|-----------|-------|
| #968 | Instrument remaining CognitionCore.tick() stages | M | Low | Observability |
| #1085 | Restore appraisal weight enrichment | S | Low | Rebase artifact |
| #1086 | PersonalityMetadataEnricher orphaned | XS | Low | Rebase artifact |
| #1087 | Log unrecognised mode names | XS | Low | |
| #1088 | loadOrCreate TOCTOU | XS | Low | |
| #1089 | Unbounded cache growth | XS | Low | |
| #1094 | **Wire TheoryOfMindEngine into tick** | **S** | **Low** | **Recommended next** |
| #1095 | MentalModel BDI → trust evolution | S | Med | |
| #1096 | Surface Bayesian ToM posterior in prompt | XS | Low | Depends on #1094 |
| #974 | Fix eval test compilation | XS | Low | |
| #975 | Document CAPS modules in contributor guide | S | Low | |
| #1043 | Oscillation attractor strategic differentiation | S | Med | |
| #1044 | CognitiveEmergenceSampleTest distrust attractor | S | Med | |
| #976 | DriveProfileDerivationPhase personality classification | M | High | |
| #977 | PID-5 facet extension | M | High | |

## Slot Status

**Slot 196 is marked ready for archive.** All repos published. Cannot be archived from within the slot.

## Neocortex Build Note

Full reactor fails at `schema-generator`. Use targeted module builds:
```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/cognition/pom.xml test -s examples/.mvn/slot-settings.xml
```

## Slot 211 Boundary

Slot 211 owns #1023 (Drive annotation provenance) and #827 (Cognitive curiosity epic). Slot 211 landed #980 (ConsolidationScheduler journal observability). Issues #974, #975, #1043, #1044, #976, #977 are safe for either slot.
