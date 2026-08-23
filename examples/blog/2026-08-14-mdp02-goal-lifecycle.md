---
layout: post
title: "Goals from Reflection — Replacing Hand-Rolled Goal Management with Engine SPIs"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [wacky-manor]
tags: [autonomous-agents, goal-lifecycle, engine-spi, reflection, cognitive-architecture]
series: issue-41-autonomous-agent-template
---

*Continues from [Memory Stack Wiring](2026-08-11-mdp01-memory-stack-wiring.md) — the first layer of the autonomous agent template.*

The goal lifecycle is the second layer of the autonomous agent template — goals that emerge from reflection rather than being hand-coded into the LLM response.

The prior session wired neocortex memory (#42): salience-scored retrieval, relationship tracking, reflection synthesis with decay. That gave characters memory. This session gives them ambition — the ability to form, revise, and abandon goals based on what they've experienced.

The design question that framed everything: wacky-manor has no cases. The engine's goal SPIs (`GoalFormationStrategy`, `GoalRevisionStrategy`) were designed for case-managed workers, and both context records carried a `CaseDefinition` parameter — a 30-field engine model class. Passing null felt wrong. I looked at the engine implementations and found neither strategy ever reads `context.definition()`. The evaluators extract everything they need — goals, agent ID, memories — before constructing the context. The `CaseDefinition` was dead weight on an SPI boundary.

So I challenged it. We removed `CaseDefinition` from both context records (engine#897). Same session, same principle: the strategies also returned `Uni<>` (Mutiny reactive types), but both implementations wrapped synchronous code in `Uni.createFrom().item()` and both callers immediately `.await()`-ed the result. Pure ceremony. That got cleaned up too.

The third engine change was adding a `GoalRevisionAction` enum — REVISE, ABANDON, COMPLETE — to `GoalRevisionProposal.RevisedGoal` (engine#903). The decision review caught that the original design used a string prefix convention ("ABANDON:" in `revisionReason`) to signal abandonment. Claude flagged it as a workaround the design philosophy prohibits, and it was right. The same review surfaced that goal completion wasn't modelled at all — when a character achieves a goal, it should be marked complete, not abandoned. The enum handles all three lifecycle transitions in one type-safe field.

The architectural insight that made the design click: plans are System 1, goals are System 2. The existing `currentPlan` (persisted `thinking` field) already handles reactive intent — the Hooded Claw sees poison, his thinking becomes "Step 1: take it. Step 2: go to the ballroom." That's per-tick, immediate, tactical. Goals are the strategic layer that emerges from reflection — after accumulating experiences and synthesising insights, the character forms objectives that persist across many ticks. Removing `newGoals`/`dropGoals` from the LLM response format was a deliberate choice: characters don't unilaterally decide their goals. Goals emerge from what they've experienced.

The implementation was five tasks, dependency-ordered. Task 1 excised `DynamicGoal` entirely — the record, the `CharacterState` collection, the response fields, the tick loop processing, the `[SITUATIONAL]` prefix in observations. Clean removal, 211 lines deleted. Tasks 2-3 built the new components: `ManorGoalFormationStrategy` (LLM proposes goals from reflection insights) and `ManorGoalEvaluator` (orchestrates formation, validates proposals, registers updated descriptors on `AgentRegistry` with per-agent locking and tick-count cooldown). Task 4 wired it all together — the evaluator chains after reflection on the same virtual thread, extracting insight texts from `ReflectionEvent` objects. The spec review caught that I'd originally planned to expand the reflection synthesiser with a combined insights+assessment method, which would have forced the `AgentExperienceService` field type from the `ReflectionSynthesizer` interface to the concrete `ManorReflectionSynthesizer`. Claude was right to flag the SPI boundary violation — we kept the interface clean.

Task 5 landed the revision strategy after engine#903 shipped. `ManorGoalRevisionStrategy` parses REVISE, ABANDON, and COMPLETE actions from the LLM, and the evaluator applies them: revisions update the goal description, abandonment and completion remove the goal and ingest a memory event so the character remembers why.

The slot `.m2` isolation issue cost me time. Slots have `.mvn/maven.config` with `-Dmaven.repo.local=<slot>/.m2`, which redirects all Maven resolution to a separate local repository. When engine#903 was installed on the host, `mvn install` wrote to `~/.m2` — invisible to the slot. The error looked like a compilation failure ("cannot find symbol: GoalRevisionAction") even though `javap` confirmed the class existed in the jar. Maven was resolving a stale timestamped remote SNAPSHOT over the local install. The fix: install with the slot's `-Dmaven.repo.local` explicitly.

The full pipeline is now: tick → memory ingest → reflection trigger fires → synthesise insights → form goals (LLM) → revise/abandon/complete goals (LLM) → register updated descriptor → goals appear in next tick's observation. Three LLM calls chained on a single virtual thread, post-reflection. Characters act on their current plan during the 6-30 second latency. When the chain completes, the new goals influence their next observation.

What this opens up: the plan structure layer (#44) can now build on goals — structured plans that break goals into sub-objectives. Trust and personality (#45) can use goal outcomes as trust signals. The template is taking shape: memory gives characters continuity, goals give them direction, and both emerge from reflection rather than being engineered into the prompt format.
