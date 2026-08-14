---
layout: post
title: "Goals Think, Plans Act"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-examples]
tags: [cognitive-architecture, plan-structure, wacky-manor, engine-spi]
series: issue-41-autonomous-agent-template
---

*Continues from [Goal Lifecycle](2026-08-14-mdp02-goal-lifecycle.md).*

## The missing middle

After wiring goals into the manor's reflection pipeline, there was a gap. Characters had strategic objectives ("protect Penelope") and reactive per-tick reasoning ("I see poison on the shelf — grab it"), but nothing connecting the two. The `currentPlan` field — a plain string the LLM wrote into its "thinking" response — was doing double duty as both tactical decomposition and moment-to-moment scratchpad.

The question wasn't whether to add structure. It was where the boundary falls between strategic intent and tactical execution. Goals say *what*. Thinking says *right now*. Plans say *how* — and that's a different cognitive process than either.

## System 1 meets System 2

The three-layer model maps directly to dual-process theory. Thinking is System 1 — fast, reactive, per-tick. The character sees the world and responds. Goals are System 2 — slow, deliberative, emerging from reflection. Plans sit between them: structured enough to track progress, fluid enough to revise when a step fails.

Each layer has its own author, cadence, and storage:

- **Goals** form from reflection via `ManorGoalFormationStrategy`, live on `AgentDescriptor` (the agent's identity), and persist across the scenario.
- **Plans** form when goals form via `ManorPlanFormationStrategy`, live on `CharacterState` (transient execution state), and revise on failure or during reflection.
- **Thinking** is the LLM's per-tick response, stored as `currentThinking` on `CharacterState`, shown back next turn.

The old `currentPlan` string collapsed all three into one field. Separating them means the character's observation now shows structured plan steps alongside its reactive scratchpad — it can see "Plan: protect-penelope → [PENDING] Find the poison, [PENDING] Dispose of it" next to "Your Current Thinking → I see Sneekly near the kitchen, better move fast."

## Using a sealed SPI you can't extend

We implemented the engine's `PlanRevisionStrategy` SPI directly — same approach as the goal SPIs from the previous session. But the plan SPI had a surprise: `AdaptationCause` is a sealed interface. Only `StepFailed` and `StepCompleted` are permitted. No creating `ActionFailureCause` or `ReflectionCause` from outside the engine package.

The compiler told us this, unhelpfully, as "class is not allowed to extend sealed class." The `javap` output shows it as a plain `public interface` — the sealed constraint lives only in the bytecode's `PermittedSubclasses` attribute.

The fix was pragmatic: repurpose what exists. `StepFailed` carries action failures — encode the action type and target in the `stepId` field ("MOVE:kitchen"), the failure reason in `reason`. `StepCompleted` carries reflection triggers — put insights in the `output` map. Not the intended semantics, but the SPI's contract doesn't care about our field naming conventions.

The `AdaptationContext` constructor was equally opinionated — `Objects.requireNonNull` on `caseId`, `definition`, `latestStatus`, and every `capabilityName`. Fields the manor will never read. We fill them with placeholder values: a random UUID, an empty `CaseDefinition`, `TaskStatus.COMPLETED`, empty strings. The constructors pass, the strategy receives its context, and the LLM in the strategy only sees the prompt we build from the fields we actually use.

This is the cost of using an SPI designed for case execution in a case-free world. Pre-release, the cost is placeholder values and some type gymnastics. The alternative — building a parallel plan model — would have been cleaner locally but would have diverged from the engine's contract. When the engine eventually relaxes its constraints, we're already wired in.

## What plans change

Characters now decompose each goal into 2–5 concrete steps via an LLM call at formation time. When an action fails — `ActionResult.Failed` in the tick loop — the plan evaluator fires `PlanRevisionStrategy` with the failure context. When reflection runs, all plans get a strategic reassessment pass.

The interesting consequence is that the thinking field genuinely becomes reactive. Before, it was trying to be both strategic plan and immediate reasoning. Now, with plans handling the tactical layer, the response format instruction changed from "write strategy" to "write your immediate reasoning about the current situation." The LLM is freed from carrying forward its own plan — the system tracks that.

Whether characters actually produce better behaviour with this separation is an empirical question for the eval tests. The architecture says they should: clearer context (here are your plans, here are your goals, now think about what to do this turn) should produce more coherent action selection than "here's a blob of your previous thinking, figure it out."

Next up: #45 — trust and personality. The last cognitive layer before the template is complete.
