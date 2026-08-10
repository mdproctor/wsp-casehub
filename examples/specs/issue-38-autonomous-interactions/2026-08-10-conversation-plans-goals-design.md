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
    { "name": "protect-tea", "description": "Stop Sneekly from poisoning the tea" }
  ],
  "dropGoals": ["find-diamond"]
}
```

### Field Semantics

- **thinking** — persisted on CharacterState as `currentPlan`. Fed back as
  "Your Current Plan" section in the next observation. The LLM's reasoning
  becomes a multi-turn strategy that it can build on, revise, or abandon.
  The format instruction must tell the LLM: "Your thinking field is your
  persistent strategic plan. It will be shown back to you next turn as
  'Your Current Plan.' Write it as a strategy you can act on, not as
  stream-of-consciousness reasoning."
- **dialogue** — what the character says aloud. Broadcast to room when
  `talkTo` is null. Directed at a specific character when `talkTo` is set.
- **talkTo** — optional character-id. When set, dialogue is directed at that
  character. Observation rendering differentiates directed vs broadcast speech.
- **aside** — private audience commentary (existing behavior, unchanged).
- **action** — mechanical action (existing behavior + PULL_ASIDE).
- **newGoals** — LLM-generated situational goals. Conform to `AgentGoal` type
  (visibility=PRIVATE, capabilities=empty). Stored on CharacterState with
  creation tick metadata. Appear in Goals section alongside Eidos-defined goals.
- **dropGoals** — removes previously generated dynamic goals by normalized name
  (lowercase, trimmed). `["*"]` drops all situational goals as a reset valve.

### Parse Error Handling

`AgentResponse.parse()` logs all parse failures at WARN level, including the
raw LLM response text, before falling back to `idle()`. This makes parse
failures visible during LLM evaluation testing.

No partial parsing — the LLM response is a coherent unit. If the LLM produces
valid dialogue but malformed `newGoals`, discarding the entire response is
correct. Partially applying the response would create state inconsistencies
(the LLM intended the dialogue AND the goal update together). Jackson's
`@JsonIgnoreProperties(ignoreUnknown = true)` already handles missing fields
gracefully — only malformed values cause parse failure.

### Platform Extraction

The response schema pattern (static identity + dynamic goals + persistent
strategy + directed communication) is domain-independent. Extractable as a
standard autonomous agent response schema.

## Directed Dialogue

### Flow

When `talkTo` is set on the response, the orchestrator validates the target
is present in the speaker's room. If the target is not present, `talkTo` is
nulled — the dialogue becomes broadcast speech. No special feedback; the LLM
sees the "Characters Present" section next tick and self-corrects.

Valid directed dialogue events carry the target. The event is routed to
everyone in the room (speech, not telepathy) but rendered differently based
on observer capabilities.

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

PULL_ASIDE runs as a **blocking inter-tick sub-procedure**. After responses
arrive and before the next tick starts:

1. **Conflict resolution:** If multiple PULL_ASIDE actions target the same
   character, or mutual PULL_ASIDE (A targets B, B targets A), only the
   first in processing order succeeds. Others get `ActionResult.Failed`
   ("Target is already in a focused exchange"). This mirrors `tryTakeObject`
   — first-to-process wins, losers get clear feedback.
2. **Dialogue suppression:** The initiator's `dialogue` field is NOT broadcast
   as normal speech. It becomes the opening line of the exchange. During
   dialogue processing, skip any character whose action is PULL_ASIDE.
3. **Exchange execution:** Alternating LLM calls, each participant seeing the
   other's dialogue plus their own current plan. Capped at N round-trips
   (configurable, default 3). Either character ends the exchange by producing
   WAIT or any action. Non-dialogue responses are **termination signals only**
   — they do not execute. The character can take the intended action on their
   next normal tick. This keeps the exchange as pure conversation.
   Nested PULL_ASIDE (participant targets a third character during exchange)
   is treated as a termination signal like any other action.
4. **No concurrent ticks:** Other agents do not tick during the exchange.
   The world is frozen — no character moves, no objects change hands. This
   eliminates stale-state problems entirely.
5. **Exchange timeout:** Total exchange duration capped at `exchangeTimeoutSeconds`
   (configurable, default 120). If exceeded, the exchange terminates and both
   participants resume with their last thinking as their plan.
6. **Publication:** Full exchange published to observation system as directed
   dialogue events — capability-gated for room observers.

### Observer Visibility

| Observer | Observation |
|---|---|
| Perceptive (same room) | Full transcript in Keen Observations |
| Non-perceptive (same room) | "Sneekly and Muttley had a quiet conversation." |
| Not in room | Nothing |

### Participant Experience

**Target's initial observation:** When character B is pulled aside by A,
B's first exchange prompt includes:
- `"[A's name] has pulled you aside and says: '[A's initiating dialogue]'"`
- B's current plan (if any)
- Condensed room context
- Exchange-specific format instruction (see below)

**Subsequent turns:** each participant sees:
- The other's dialogue from the previous turn
- Their own current plan (so the LLM can build on its strategy)
- Condensed room context

**Condensed room context** = room name + names of other characters present.
Enough to stay grounded in the physical space without the full observation
rebuild (no objects, exits, inventory, affordances).

**Exchange format instruction:** the exchange prompt replaces the normal
response format instruction with a simplified version: respond with JSON
containing `thinking`, `dialogue`, and optionally `action`. The instruction
explicitly states: "Respond with WAIT to end this private conversation and
return to normal activity." This disambiguates WAIT (end exchange) from its
normal-tick meaning (do nothing this turn).

The `thinking` field from each turn overwrites `currentPlan`. This IS
accumulation — through LLM synthesis, not mechanical concatenation. Because
the LLM sees its current plan each turn, each new thinking incorporates and
updates the prior plan. The overwrite is correct because the new thinking
subsumes whatever the LLM retained from the prior plan.

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
   containing the previous tick's thinking. When `currentPlan` is null
   (first tick), the section is omitted entirely — consistent with how
   `keenObservationsSection` handles empty state.
4. The LLM sees its own prior reasoning and builds on it

### What This Enables

- HC: "Step 1: get poison. Step 2: approach tea. Step 3: wait for distraction"
  — executed across ticks
- Mob: "Something ain't right about Sneekly. Watch him near Penelope"
  — maintained across room changes
- Dastardly: "Get the map, give Peter wrong directions, take the real path"
  — multi-step heist

### PULL_ASIDE Integration

Both participants see their current plan each exchange turn (see §Participant
Experience). Each turn's thinking overwrites `currentPlan`. Because the LLM
sees its prior plan, it incorporates and revises it — accumulation through
synthesis, not concatenation. The pre-exchange plan is not lost unless the
LLM chooses to abandon it.

### Memory Integration

Current plan ingested into neocortex memory when it changes — determined by
**string inequality** against the previous tick's plan text. Simple,
deterministic, cheap. Identical plans across ticks (character repeated its
thinking verbatim) don't re-ingest. Provides strategic context for
reflection when it arrives on the platform.

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
  tick for eviction ordering. `DynamicGoal` is a wacky-manor model record
  (`String name, String description, int creationTick`), not an Eidos
  `AgentGoal`. ObservationBuilder renders both types in the Goals section
  but from separate collections — `List<AgentGoal>` for identity goals,
  `List<DynamicGoal>` from CharacterState for situational goals.
- **Name uniqueness:** per-character. If the LLM creates a goal with a name
  that already exists on that character, the existing goal is replaced
  (update semantics). This handles the LLM naturally refining a goal.
- **Visibility:** dynamic goals are always private to the owning character.
  Other characters cannot see them. Consistent with the spec's intent —
  they are internal situational assessments.
- **Processing order:** `dropGoals` is processed before `newGoals` within the
  same response. This allows replacing a goal in one response (drop old name,
  create new). Dropping a name that matches no dynamic goal is silently ignored.
- **Tier guard:** `dropGoals` only operates on dynamic (situational) goals.
  Names matching identity-tier goals are silently ignored. The rendering
  distinguishes `[PRIMARY]`/`[SECONDARY]` from `[SITUATIONAL]`, making it
  clear to the LLM which goals are droppable.
- `ObservationBuilder` renders both tiers in Goals, visually distinguished:
  `[PRIMARY] Find the Doily Diamond` vs `[SITUATIONAL] Protect the tea`
- Dynamic goals ingested into neocortex memory when created. **Goal drops
  are also memory events** — "Abandoned goal: [name]" is ingested, recording
  the strategic decision to abandon a goal.
- Dynamic goal storage on `CharacterState` uses `CopyOnWriteArrayList`,
  consistent with the inventory collection's concurrency discipline

### Goal Cap

Dynamic goals are capped at `maxDynamicGoals` (configurable, default 5).
When `newGoals` would exceed the cap, the oldest dynamic goals (by creation
tick) are auto-evicted to make room. The LLM controls what stays by actively
dropping stale goals before the cap forces eviction.

The creation tick stored on each `DynamicGoal` serves eviction ordering —
oldest-first when the cap is reached.

## Behavioral Assertions (Replacing Hardcoded Completion)

### Completion

- Remove `hasEffect("tea-service", "rat-poison")` as a game-ending trigger
- Remove `CompletionReason.POISONED` — without a CONSUME action type, there
  is no mechanical definition of "tea consumed" and the completion reason is
  unreachable
- Game completion is purely time-based: configurable `maxTurns` representing
  dawn
- Rename `CompletionReason.TURN_LIMIT` to `CompletionReason.DAWN`
- Whether poisoning behavior occurred is tracked by behavioral assertions,
  not by game completion — the engine observes, it doesn't decide outcomes

### Assertions

Behavioral assertions are `Predicate<TickSnapshot>` instances registered at
scenario start. The orchestrator evaluates all registered assertions after
each tick and logs results via structured logging (marker: `ASSERTION`).

Example assertions:
- "HC generated a poison-related goal"
- "HC attempted STEAL or USE with poison"
- "Mob intervened when a danger was observed"
- "Peter reacted protectively to a danger observation"

`AssertionRegistry` provides a query API for eval tests:
- `boolean wasSatisfied(String assertionId)` — ever true during the scenario
- `int firstSatisfiedTick(String assertionId)` — tick when first satisfied
- `List<AssertionResult> history(String assertionId)` — per-tick results

Observable, not controlling. Used for LLM evaluation and tuning, not game
logic. The assertion framework is a platform extraction target — definition
format and query API are designed for reuse.

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
