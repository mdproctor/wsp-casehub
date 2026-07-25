# Goals and Constraints on AgentDescriptor

**Issue:** casehubio/eidos#100
**Date:** 2026-07-25
**Status:** Approved

## Problem

Agent goals are embedded in the `briefing` text field as natural language.
This prevents programmatic goal visibility control, goal-aware rendering,
and structured constraint enforcement. Goals like "find the diamond" and
constraints like "never break cover" should be first-class descriptor fields
with explicit visibility, priority, and format-discriminated rendering.

## Design Decisions

- **Goals and aims are the same concept** — both collapse into `AgentGoal`
- **Beliefs are out of scope** — pre-loaded knowledge intersects RAG; separate issue
- **No goal-based querying** — goals are identity + rendering only; `AgentQuery` unchanged
- **BDI as orientation vocabulary, not execution architecture** — goals/constraints
  structure the system prompt that bootstraps the LLM; the LLM does the reasoning
- **`AgentGoal` (standing, descriptor) and `GoalContext` (current, prompt context)
  coexist** — same concept at different time scales; no rename needed
- **Rendering order: goals/constraints after capabilities, before disposition** —
  motivation before behavioral style

## Data Model

### AgentGoal (Tier 1, api/)

```java
public record AgentGoal(
    String name,           // unique within descriptor, <= 100 chars
    String description,    // human/LLM-readable, <= 500 chars
    GoalPriority priority, // PRIMARY or SECONDARY
    Visibility visibility  // PUBLIC or PRIVATE
) {}
```

Validated in compact constructor: name required, description required,
priority required, visibility required. No control characters.
Constants: `MAX_GOAL_NAME = 100`, `MAX_GOAL_DESCRIPTION = 500`.

### GoalPriority (Tier 1, api/)

```java
public enum GoalPriority { PRIMARY, SECONDARY }
```

Two levels. PRIMARY = main objectives. SECONDARY = supporting objectives.
Equal-priority goals expressed by giving multiple goals the same level.

### AgentConstraint (Tier 1, api/)

```java
public record AgentConstraint(
    String name,           // unique within descriptor, <= 100 chars
    String description,    // human/LLM-readable, <= 500 chars
    Visibility visibility  // PUBLIC or PRIVATE
) {}
```

Validated in compact constructor: name required, description required,
visibility required. No control characters.
Constants: `MAX_CONSTRAINT_NAME = 100`, `MAX_CONSTRAINT_DESCRIPTION = 500`.

### Visibility (Tier 1, api/)

```java
public enum Visibility { PUBLIC, PRIVATE }
```

Shared enum for both goals and constraints. PUBLIC items appear in all
render formats including A2A_CARD. PRIVATE items appear only in the
owning agent's system prompt (MARKDOWN/PROSE) and are completely absent
from A2A_CARD — not paraphrased, not hinted at, omitted entirely.

### AgentDescriptor Changes

Two new fields:

```java
public record AgentDescriptor(
    // ... existing 18 fields ...
    List<AgentGoal> goals,
    List<AgentConstraint> constraints
) {}
```

Both nullable in constructor, defaulting to `List.of()`.

Builder gains `.goals(List<AgentGoal>)` and `.constraints(List<AgentConstraint>)`.

Convenience methods:
- `publicGoals()` — filters by `Visibility.PUBLIC`
- `publicConstraints()` — filters by `Visibility.PUBLIC`

Validation in compact constructor:
- Goal names unique within the descriptor
- Constraint names unique within the descriptor

## Rendering

### Prompt Section Order

1. Header (name, model, provider)
2. Role (slot)
3. Capabilities
4. **Goals** (NEW)
5. **Constraints** (NEW)
6. How You Operate (disposition)
7. Operating Principles (briefing)
8. Data Handling
9. Current Goal (from GoalContext)
10. Resources
11. Context

### Format-Specific Rendering

**MARKDOWN:**
```markdown
## Goals
- **[PRIMARY]** Find the Doily Diamond
- **[SECONDARY]** Help other treasure hunters

## Constraints
- You see the best in everyone and trust them by default
- You do not notice when you are in personal danger
```

**PROSE:**
Goals as flowing sentence ("Your primary objectives are X and Y.
You also aim to Z."). Constraints as behavioral directives
("You must always... You must never...").

**A2A_CARD (JSON):**
```json
{
  "goals": [
    {"name": "win-treasure", "description": "Win the treasure hunt",
     "priority": "SECONDARY"}
  ],
  "constraints": [
    {"name": "elaborate-schemes",
     "description": "Your schemes must be elaborate and theatrical"}
  ]
}
```
Only PUBLIC items. PRIVATE items completely absent.

### Enrichment Pipeline

Goals and constraints render **structurally always**. They are NOT sent
to the LLM enrichment step. Rationale:
- Goals are already natural language — no axis-to-prose conversion needed
- Constraints are behavioral guardrails — LLM rephrasing risks softening
  or losing critical wording

The existing enrichment step continues to handle disposition (axis → prose)
and current goal (GoalContext → narrative).

## YAML Format

```yaml
descriptors:
  - agentId: hooded-claw
    name: The Hooded Claw
    slot: villain
    tenancyId: wacky-manor
    goals:
      - name: eliminate-penelope
        description: "Kill Penelope Pitstop before she finds the treasure"
        priority: PRIMARY
        visibility: PRIVATE
      - name: win-treasure
        description: "Win the treasure hunt"
        priority: SECONDARY
        visibility: PUBLIC
    constraints:
      - name: never-break-cover
        description: "Never reveal your true identity as The Hooded Claw"
        visibility: PRIVATE
      - name: elaborate-schemes
        description: "Your schemes must be elaborate and theatrical"
        visibility: PUBLIC
```

New inner config classes `GoalConfig` and `ConstraintConfig` in
`ClasspathYamlDescriptorRegistrar`.

## JPA Persistence

Flyway V7 — two new child entity tables:

### agent_goal

| Column | Type | Constraint |
|---|---|---|
| id | BIGINT | PK, auto-generated |
| descriptor_id | BIGINT | FK → agent_descriptor(internal_id), NOT NULL |
| agent_id | VARCHAR(255) | NOT NULL |
| tenancy_id | VARCHAR(255) | NOT NULL |
| name | VARCHAR(100) | NOT NULL |
| description | TEXT | NOT NULL |
| priority | VARCHAR(20) | NOT NULL |
| visibility | VARCHAR(20) | NOT NULL |

### agent_constraint

| Column | Type | Constraint |
|---|---|---|
| id | BIGINT | PK, auto-generated |
| descriptor_id | BIGINT | FK → agent_descriptor(internal_id), NOT NULL |
| agent_id | VARCHAR(255) | NOT NULL |
| tenancy_id | VARCHAR(255) | NOT NULL |
| name | VARCHAR(100) | NOT NULL |
| description | TEXT | NOT NULL |
| visibility | VARCHAR(20) | NOT NULL |

Entity classes `AgentGoalEntity` and `AgentConstraintEntity` follow the
`AgentCapabilityEntity` pattern: `@OneToMany(mappedBy, cascade=ALL,
orphanRemoval=true)` on `AgentDescriptorEntity`.

`AgentDescriptorMapper` gains `toGoal`/`toGoalEntity` and
`toConstraint`/`toConstraintEntity` methods.

## Comparator

`AgentDescriptorComparator` gains `compareGoals()` and `compareConstraints()`
methods following the `compareCapabilities()` pattern:

- Keyed by name
- Detect present/absent drift
- Field-level drift within matching entries

Constants:
- `COMPARED_FIELD_COUNT` incremented by 2
- `COMPARED_GOAL_FIELD_COUNT = 3` (description, priority, visibility)
- `COMPARED_CONSTRAINT_FIELD_COUNT = 2` (description, visibility)

## What This Does NOT Change

- `AgentQuery` — no goal-based querying
- `AgentRegistry.find()` — no filtering by goals
- `GoalContext` / `AgentPromptContext` — unchanged
- `CapabilityHealth` / `BehavioralSignalStore` — goals are identity, not health
- `VocabularyRegistry` — no vocabulary grounding for goals/constraints
- Enrichment pipeline — goals/constraints are structural, not enriched

## Test Coverage

- `AgentGoal` / `AgentConstraint` compact constructor validation (nulls, blanks, length, control chars)
- `AgentDescriptor` goal/constraint name uniqueness validation
- `AgentDescriptor.publicGoals()` / `publicConstraints()` filtering
- YAML loading with goals and constraints
- YAML loading with missing/empty goals and constraints (backward compat)
- MARKDOWN rendering with goals and constraints sections
- PROSE rendering with goals and constraints
- A2A_CARD rendering — PUBLIC only, PRIVATE absent
- `AgentDescriptorComparator` drift detection for goals and constraints
- JPA round-trip: persist and retrieve goals and constraints
- Existing tests remain green (goals/constraints default to empty lists)
