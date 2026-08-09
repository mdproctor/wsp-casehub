# Autonomous Conversation, Persistent Plans, and Dynamic Goals — Design

**Issue:** casehubio/examples#38
**Date:** 2026-08-10
**Builds on:** 2026-08-09-autonomous-interactions-design.md (capability-gated observations, concealment, STEAL)

## Design Principle

LLM-driven autonomy by default. Mechanical support only where the LLM
consistently fails without it. Every pattern designed for platform extraction
into Eidos, Neocortex, Blocks, or Engine.

## Response Format

The LLM response expands from 4 fields to 7:

```json
{
  "thinking": "internal reasoning — persisted as current plan",
  "dialogue": "what you say aloud (or null)",
  "talkTo": "character-id for directed dialogue (or null for broadcast)",
  "aside": "private thoughts for the audience (or null)",
  "action": { "type": "MOVE", "target": "kitchen", "withItem": null },
  "newGoals": [
    { "name": "protect-tea", "description": "Stop Sneekly from poisoning the tea", "priority": "PRIMARY" }
  ],
  "dropGoals": ["find-diamond"]
}
```

### Field Semantics

- **thinking** — persisted on CharacterState as `currentPlan`. Fed back as
  "Your Current Plan" section in the next observation. The LLM's reasoning
  becomes a multi-turn strategy that it can build on, revise, or abandon.
- **dialogue** — what the character says aloud. Broadcast to room when
  `talkTo` is null. Directed at a specific character when `talkTo` is set.
- **talkTo** — optional character-id. When set, dialogue is directed at that
  character. Observation rendering differentiates directed vs broadcast speech.
- **aside** — private audience commentary (existing behavior, unchanged).
- **action** — mechanical action (existing behavior + PULL_ASIDE).
- **newGoals** — LLM-generated situational goals. Registered on CharacterState,
  appear in Goals section alongside Eidos-defined goals.
- **dropGoals** — removes previously generated dynamic goals by name.

### Platform Extraction

The response schema pattern (static identity + dynamic goals + persistent
strategy + directed communication) is domain-independent. Extractable as a
standard autonomous agent response schema.

## Directed Dialogue

### Flow

When `talkTo` is set on the response, the dialogue event carries the target.
The event is routed to everyone in the room (speech, not telepathy) but
rendered differently based on observer capabilities.

### ManorEvent Changes

`ManorEvent` gets `String dialogueTarget` field (null for broadcast dialogue).

### NarrativeEventBuilder

Directed dialogue renders as:
- Public (non-perceptive): `"Sneekly spoke quietly with Peter."`
- Detailed (perceptive / participants): `"Sneekly, speaking to Peter: 'The cellar is perfectly safe, I assure you.'"`

This reuses the `detailedDescription` / keen observations pattern from
capability-gated observations. `ManorEvent.description` carries the vague
version, `detailedDescription` carries the full dialogue.

### Capability-Gated Overhearing

| Observer | What they see |
|---|---|
| Has `perception` tag | Full dialogue in Keen Observations |
| No `perception` tag | "Sneekly spoke quietly with Peter." in Recent Activity |
| Is the `talkTo` target | Full dialogue (always) |
| Is the speaker | Full dialogue (always) |

### Deception in Conversation

LLM-driven by default. The character's personality, goals, and constraints
determine whether they lie. Perceptive observers may notice inconsistencies
because their descriptor tells them to be suspicious, not because the engine
flags lies.

If testing reveals the LLM consistently fails to detect lies that perceptive
characters should catch, add mechanical hints: when a character with
`deception` capability tag speaks directed dialogue, perceptive observers
get a keen observation hint ("You sense Sneekly is not being entirely
truthful"). This is a fallback, not the default design.

## Focused Exchanges (PULL_ASIDE)

### Action

```json
{ "action": { "type": "PULL_ASIDE", "target": "muttley" },
  "dialogue": "Muttley, I need you to do something for me..." }
```

### Engine Behavior

1. Orchestrator pauses initiator and target from normal tick cycle
2. Runs focused mini-conversation: alternating LLM calls, each seeing the
   other's response. Capped at N round-trips (configurable, default 3)
3. Either character ends the exchange by producing a non-PULL_ASIDE action
   or WAIT
4. Full exchange published to observation system as directed dialogue events —
   capability-gated for room observers

### Observer Visibility

| Observer | Observation |
|---|---|
| Perceptive (same room) | Full transcript in Keen Observations |
| Non-perceptive (same room) | "Sneekly and Muttley had a quiet conversation." |
| Not in room | Nothing |

### Participant Experience

Each turn of the exchange, the participant sees the other's dialogue plus
condensed room context (not a full observation rebuild). The `thinking` field
from each turn accumulates into their persistent plan.

### Resolution

After the exchange ends, both characters resume normal tick participation.
Exchange content ingested into neocortex memory for both participants.

### Platform Extraction

Maps to Qhorus QUERY/RESPONSE commitment pattern. Initiator sends QUERY,
target responds, threaded via `inReplyTo` and `correlationId`. Reusable
"agent conversation" pattern: two agents, directed channel, capped turns,
commitment resolution.

## Persistent Plans

### Mechanism

1. `CharacterState` gets `String currentPlan` field
2. After each tick, `AgentResponse.thinking()` stored as
   `character.setCurrentPlan(thinking)`
3. `ObservationBuilder` adds "Your Current Plan" section (before Goals),
   containing the previous tick's thinking
4. The LLM sees its own prior reasoning and builds on it

### What This Enables

- HC: "Step 1: get poison. Step 2: approach tea. Step 3: wait for distraction"
  — executed across ticks
- Mob: "Something ain't right about Sneekly. Watch him near Penelope"
  — maintained across room changes
- Dastardly: "Get the map, give Peter wrong directions, take the real path"
  — multi-step heist

### PULL_ASIDE Integration

Both participants' thinking accumulates during focused exchanges — the plan
updates with what they learned from conversation.

### Memory Integration

Current plan ingested into neocortex memory when it substantively changes
(not every tick). Provides strategic context for reflection when it arrives
on the platform.

### Platform Extraction

"Persistent agent reasoning" — any autonomous agent benefits from seeing its
own prior reasoning. Simple implementation (one field on state, one
observation section), large behavioral impact (reactive → strategic).

## Dynamic Goals

### Three Tiers

| Tier | Source | Mutable | Example |
|---|---|---|---|
| Identity | Eidos descriptor | No | "Eliminate Penelope before dawn" |
| Situational | LLM via `newGoals` | Yes (`dropGoals`) | "Protect the tea" |
| Tactical | Implicit in plan | No registration | "Get to ballroom first" |

### Mechanical Lifecycle

- `newGoals` creates `DynamicGoal` records on `CharacterState` with creation
  tick for staleness tracking
- `dropGoals` removes by name
- `ObservationBuilder` renders both tiers in Goals, distinguished:
  `[PRIMARY] Find the Doily Diamond` vs `[SITUATIONAL] Protect the tea`
- Dynamic goals ingested into neocortex memory when created

### No Goal Limit

The LLM sees all goals every tick and decides what matters. If it generates
too many, its own reasoning deprioritizes or drops stale ones.

## Behavioral Assertions (Replacing Hardcoded Completion)

### Completion

- Remove `hasEffect("tea-service", "rat-poison")` as a game-ending trigger
- Game completion becomes time-based: configurable `maxTurns` representing dawn
- New `CompletionReason.DAWN` alongside existing `TURN_LIMIT`
- `POISONED` remains as a completion reason if tea IS poisoned AND consumed,
  but the engine doesn't check for a specific effect — characters drive the
  outcome

### Assertions

Behavioral assertions checked each tick and logged as metrics:
- "HC generated a poison-related goal"
- "HC attempted STEAL or USE with poison"
- "Mob intervened before tea was consumed"
- "Peter reacted protectively to a danger observation"

Observable, not controlling. Used for LLM evaluation and tuning, not game logic.

### Platform Extraction

Behavioral assertion framework — define expected autonomous behaviors,
measure whether agents exhibit them, without constraining the outcome.
Reusable for any autonomous agent evaluation.

## Implementation Layers

Dependency-ordered:

1. **Response format** — expand AgentResponse, update parsing and format instruction
2. **Persistent plans** — currentPlan on CharacterState, ObservationBuilder section
3. **Dynamic goals** — DynamicGoal model, newGoals/dropGoals lifecycle, ObservationBuilder rendering
4. **Directed dialogue** — talkTo field, ManorEvent dialogueTarget, capability-gated overhearing
5. **PULL_ASIDE action** — ActionType, orchestrator exchange loop, observation publishing
6. **Behavioral assertions** — replace hardcoded completion, assertion framework
7. **LLM eval tests** — validate each pattern end-to-end

## Testing Strategy

### Unit Tests (per layer)
- Response parsing with new fields
- Plan persistence across ticks
- Dynamic goal lifecycle (create, render, drop)
- Directed dialogue event routing and capability-gated visibility
- PULL_ASIDE exchange mechanics (cap, termination, observation publishing)
- Behavioral assertion logging

### LLM Eval Tests
- Same scenario, character with plan vs without — verify multi-step execution
- Dynamic goal generation from observation (character sees danger → generates protective goal)
- Directed dialogue — character talks TO a specific character, not broadcast
- PULL_ASIDE — two characters exchange information, verify both act on it
- Deception — deceptive character lies in conversation, verify perceptive observer reacts differently

## Design Constraints

### Platform Extraction Readiness

Every pattern should be implementable in wacky-manor first, then extractable:
- Persistent plans → Neocortex or Engine agent state
- Dynamic goals → Eidos goal lifecycle
- Directed dialogue → Qhorus directed messaging
- PULL_ASIDE → Qhorus commitment-based conversation
- Behavioral assertions → Engine evaluation framework
- Capability-gated overhearing → Blocks observation filtering

### Autonomous LLM Template

The sum of these patterns (capability-gated perception, concealment, directed
conversation, persistent plans, dynamic goals, behavioral assertions)
constitutes a reusable template for autonomous LLM applications. The
wacky-manor implementation should draw clean lines between domain-specific
wiring and domain-independent patterns to facilitate future extraction.
