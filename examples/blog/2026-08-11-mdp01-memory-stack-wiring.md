---
layout: post
title: "The cognitive stack belongs to the agent, not the engine"
date: 2026-08-11
entry_type: note
subtype: diary
projects: [casehub-examples]
tags: [neocortex, memory, reflection, autonomous-agents, architecture]
---

Issue #41 is the big one: wire the full neocortex and engine cognitive stack
into wacky-manor as the canonical autonomous agent template. Characters
should remember, reflect, track relationships, and eventually form goals
from experience rather than from per-turn LLM output. The platform has all
of this shipped. The manor uses almost none of it.

I expected the first question to be "which API do I call?" It turned out to
be "which execution model owns the cognitive loop?"

## The tension

The engine is a process orchestration system. Its cognitive services --
goal formation, plan revision, reflection triggering -- are designed around
Workers and CaseInstances. A Worker executes a step, produces a
WorkerOutcome, and the outcome triggers the cognitive cycle: record
experience, evaluate reflection threshold, form goals from insights, revise
plans on failure.

The manor is a real-time game loop. Characters perceive the world through
observations, invoke an LLM, and act -- every tick, continuously. There are
no Workers. There is no CaseInstance. The LLM *is* the cognitive engine;
the platform services provide structure around its decisions.

Forcing the manor into the engine's Worker model would mean modelling each
tick as a "step" in a "plan" executed by a "worker" -- fighting the
engine's design at every turn. But ignoring the engine entirely would mean
re-implementing its cognitive concepts from scratch, with inevitable drift.

## The resolution

The cognitive capabilities are the platform value. The execution model is
context-specific.

The engine's public API layer (`casehub-engine-api`) provides clean SPIs
for goal formation, goal revision, and plan adaptation. The data models
(`GoalFormationContext`, `GoalRevisionProposal`) are well-designed
contracts. But the *evaluators* -- the thin orchestration that decides
*when* to call the SPIs -- are coupled to `WorkerOutcome` and
`CaseInstance`. They're internal classes, not public API.

So the architecture is three layers: neocortex for memory (clean public
APIs, no tension), engine SPIs for cognitive contracts (use them directly),
and manor-specific orchestration for the glue that decides when to trigger
reflection or form goals. The manor implements ~50 lines per evaluator.
When the engine generalises these to accept outcomes from any execution
model, the manor swaps to the shared implementation. The SPIs are already
the right contract.

## What the decision review caught

Claude ran a light adversarial review on the decisions before the spec was
written. Two findings changed the design:

`ReflectionTriggerConfig` from `casehub-engine-api` looked like a clean
configuration type. Its `importanceWeights` map is keyed by `SUCCESS`,
`COMPLETED`, `DECLINED`, `FAILED`, `EXPIRED` -- Worker outcome variant
names. The manor's action results are `Success`, `Failed`, `MovedToRoom`,
`ItemReceived`. No mapping exists. The "standardised configuration" was
semantically coupled to Workers all along. We dropped it and used
application.properties instead.

`GoalFormationContext` requires a `CaseDefinition` parameter. By depending
on this type, the manor implicitly commits to being (or pretending to be)
a CaseHub case -- which contradicts the entire rationale for avoiding the
engine's execution model. We deferred `casehub-engine-api` as a dependency
until the goal lifecycle phase, where the `CaseDefinition` question must
be resolved head-on.

Both findings point to the same platform issue: the cognitive SPIs need a
standalone module that doesn't require `CaseDefinition`. The manor's
implementation is the proof-of-concept for that extraction.

## What was wired

Five commits, each independently testable:

**Salience-scored retrieval.** `MemoryOrder.SALIENCE` replaces
`CHRONOLOGICAL`. Experience records carry importance scores mapped from
action types -- a STEAL (0.9) outranks a MOVE (0.3) in recall, even if
the move happened more recently. Without importance scoring, salience
degrades to pure recency, which is barely better than chronological.

**Relationship tracking.** When a character interacts with another (dialogue,
GIVE, STEAL, PULL_ASIDE), the experience record carries
`ExperienceAttributeKeys.TARGET_AGENT`. The neocortex's
`RelationshipObserver` automatically creates per-pair relationship
memories. No additional wiring needed -- CDI event propagation handles it.

**Reflection.** `ManorReflectionSynthesizer` implements the neocortex
`ReflectionSynthesizer` SPI. After a character accumulates enough
experiences (tracked by `ManorReflectionTrigger` -- count-based or
cumulative-importance threshold), an async virtual thread runs the
reflection loop: query salience-scored memories, call the LLM for pattern
synthesis, store the insights in the reflection domain. The reflection loop
is implemented directly rather than through the neocortex's
`ReflectionService`, which requires CDI `Event<ReflectionRecorded>` -- not
available in the manor's manual component wiring.

**Enhanced observations.** Characters now see an "Insights" section with
their reflection output and "About [Character]" sections with relationship
memories for agents in the room. These appear alongside the existing
world-state observations, giving the LLM structured context about what the
character has learned and who they've interacted with.

**Memory decay.** After each reflection cycle, old low-importance memories
are purged via `MemoryRetentionPolicy`. Reflection synthesises raw
experience into insights; the raw memories can then fade. Insights persist.

## What comes next

The goal lifecycle phase replaces `DynamicGoal` (a trivial record) and
`currentPlan` (a String field) with engine-backed goal formation, revision,
and abandonment. That's where the `CaseDefinition` question lands -- and
where the engine's public SPIs get consumed for real.
