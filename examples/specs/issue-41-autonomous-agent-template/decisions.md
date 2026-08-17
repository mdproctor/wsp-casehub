# Decisions — #41 Autonomous Agent Template

## D1: PlanRevisionStrategy SPI usage

**Choice:** Use the engine's PlanRevisionStrategy SPI directly — fill case-specific fields (caseId, compoundId, CaseDefinition, latestBindingName) with nulls. Set capabilityName on PlanStepDescriptor/CompletedStep to null or a generic placeholder. No wrapper or abstraction layer.
**Alternatives:**
- Challenge the SPI upstream (engine issue to decouple from cases) — too many fields to remove, disproportionate risk to load-bearing case execution model
- Manor-local plan model (skip SPI entirely) — loses the SPI contract and diverges from engine pattern
**Rationale:** Pre-release — the leaky case fields cost nothing. The SPI's core concept (given completed/pending steps and a cause, propose revised steps) is exactly what the manor needs. Wrapping to hide case details is a cheap add later if needed. Follows the same pattern as goal SPIs: implement directly, don't over-abstract.
**Trade-offs:** Manor code sees case-coupled types (AdaptationContext, CompletedStep.capabilityName). Acceptable at pre-release — wrapper deferred.
**Exploration:** quick
**Status:** captured

## D2: Plan relationship to thinking field

**Choice:** Separate concerns — keep "thinking" as a free-text per-tick scratchpad (reactive layer). Add a separate structured plan model (tactical layer) that decomposes goals into named steps with status tracking. Plans form when goals change (event-driven, not per-tick). Three-layer cognitive model: goals (strategic) → plans (tactical) → thinking (reactive).
**Alternatives:**
- Replace thinking with structured output — loses the free-text scratchpad valuable for reactive reasoning. Forces all cognition into structured steps.
- Structured thinking (hybrid) — embeds structure inside free text, creates impedance mismatch with PlanRevisionStrategy SPI which operates on clean PlanStepDescriptor lists. Couples reactive and tactical layers in one field.
**Rationale:** The SPI (D1) expects structured plan data — mixing it into a free-text blob requires extraction/injection parsing. Clean separation means plan formation/revision operates on a first-class model while thinking stays independent. Plan formation triggers on goal changes (not every tick), consistent with the goal lifecycle pattern from #43.
**Trade-offs:** Adds a plan formation LLM call when goals change. Acceptable — it's event-driven, not per-tick.
**Depends on:** D1 (SPI usage — the SPI's structured input/output model drives the separation)
**Exploration:** quick
**Status:** captured

## D3: Plan storage location

**Choice:** CharacterState — replace `String currentPlan` with a structured plan model. Plans are transient execution state (per-scenario, in-memory), same as lastActionResult and currentRoom.
**Alternatives:**
- AgentDescriptor (alongside goals) — plans aren't identity, they're execution state. Heavyweight updates (full descriptor rebuild + re-register on every step completion).
- Separate store (ConcurrentHashMap) — adds complexity for no benefit over CharacterState, which is already per-agent mutable state.
**Rationale:** Plans decompose goals into steps for the current scenario run. They don't persist across scenarios. CharacterState is exactly this: ephemeral per-agent state. ObservationBuilder already reads from CharacterState. Replacing currentPlan in-place is the minimal change.
**Trade-offs:** Goals (AgentDescriptor) and plans (CharacterState) live in different stores. The evaluator already cross-references both — not a new pattern.
**Depends on:** D2 (separate concerns — plans are a distinct model, not embedded in thinking)
**Exploration:** quick
**Status:** captured

## D4: Plan granularity

**Choice:** Per-goal plans — each goal gets its own plan (list of steps). One plan per goal, formed when the goal forms, revised when steps fail, removed when the goal completes or is abandoned.
**Alternatives:**
- Unified plan (one per character spanning all goals) — more holistic but harder lifecycle (which steps belong to which goal?). Doesn't map to AdaptationContext.goalName.
- Active-goal plan only — too restrictive; characters have multiple concurrent goals. Secondary goals would be planless.
**Rationale:** Natural extension of the goal model (goals are independent, plans mirror this). Direct SPI mapping (AdaptationContext.goalName). Clean lifecycle tied to goal lifecycle. Cross-goal reasoning happens in the thinking field (reactive layer), not the plan structure.
**Trade-offs:** No cross-goal optimization in plans. Acceptable — the LLM handles opportunistic cross-goal reasoning in the thinking field.
**Depends on:** D2 (plans separate from thinking), D3 (plans on CharacterState)
**Exploration:** quick
**Status:** captured

## D5: Plan revision triggers

**Choice:** Both — immediate revision on action failure (reactive) plus reflection-driven revision for strategic re-assessment (deliberative). Two triggers, different granularity.
**Alternatives:**
- Action-outcome mismatch only — misses strategic re-assessment (step succeeded but plan is no longer viable due to changed circumstances)
- Reflection-driven only — too slow for obvious failures (took poison, but someone saw you — plan should revise immediately, not wait for reflection)
**Rationale:** Mirrors the System 1 / System 2 split. Reactive revision catches immediate failures (action failed, step blocked). Deliberative revision during reflection catches strategic obsolescence (world changed, goal context shifted). Both call PlanRevisionStrategy but with different AdaptationCause values.
**Trade-offs:** More PlanRevisionStrategy calls (one per failure + one per reflection). Acceptable — plan revision is a single LLM call, and failures are infrequent.
**Depends on:** D1 (SPI usage), D4 (per-goal plans)
**Exploration:** quick
**Status:** captured

## D6: Plan formation mechanism

**Choice:** LLM-driven via a manor-local ManorPlanFormationStrategy. Separate LLM call triggered after goal formation. Given a goal + character context + memories, the LLM decomposes the goal into named steps. Consistent with ManorGoalFormationStrategy pattern.
**Alternatives:**
- Inline with goal formation (expand GoalFormationProposal to include plan steps) — fewer LLM calls but couples goal and plan formation. Plan revision would still need a separate mechanism, so the asymmetry isn't worth the savings.
**Rationale:** Goals and plans are separate cognitive processes (D2). Goal formation asks "what should I pursue?" Plan formation asks "how do I pursue this goal?" Different questions, different prompts, different LLM calls. Consistent with the pattern established by ManorGoalFormationStrategy and ManorGoalRevisionStrategy.
**Trade-offs:** Additional LLM call per new goal. Acceptable — goal formation is infrequent (cooldown-gated, reflection-triggered).
**Depends on:** D2 (separate concerns), D4 (per-goal plans)
**Exploration:** quick
**Status:** captured

## D7: Step status tracking

**Choice:** Reflection-assessed — step completion/failure status is assessed during reflection by the LLM, piggybacking on synthesis (same pattern as GoalOutcomeCounts from #43). No per-tick mechanical matching or response format changes.
**Alternatives:**
- Per-tick LLM assessment (structured output addition to every response) — adds parsing to the hot path, most ticks won't complete any step, changes response format
- Mechanical heuristic (match action type+target to step) — brittle, plan steps are natural language, actions are structured; multi-step actions can't be matched
**Rationale:** Step status serves the system (AdaptationContext), not the character's per-tick reasoning. The character reasons about plan progress in the thinking field using observations. Reflection already has accumulated context (memories, actions) and is already making LLM calls. Reactive revision (D5) is triggered by action failure — the failure is the signal, the LLM in PlanRevisionStrategy interprets which steps to revise.
**Trade-offs:** Step status updates lag behind real actions (delayed until reflection). Acceptable — the character's behavior is driven by the thinking field, not structured step status.
**Depends on:** D5 (revision triggers — reactive uses failure context, not step status)
**Exploration:** quick
**Status:** captured

# Decisions — #45 Trust and Personality

## D8: Trust computation method

**Choice:** Interaction-history based — count positive vs negative interactions from relationship memories. Classify memory content by keywords (help/protect/warn → positive; lie/steal/betray/trick/trap → negative). Score normalized to 0.0–1.0, default 0.5 for unknown agents.
**Alternatives:**
- LLM-judged — ask LLM to score trustworthiness after each interaction. Richer but adds LLM calls per tick per character pair.
- Signal-derived — derive trust from BehavioralSignal SUCCESS/DECLINE counts. Mechanical, no LLM cost, but less nuanced — doesn't distinguish who the action was directed at.
**Rationale:** Uses relationship memory already wired in #42. No extra LLM calls. Trust degrades naturally as characters remember lies and betrayals. Keyword matching is sufficient for Wacky Races characters where deception is theatrical and explicit.
**Trade-offs:** Keyword-based classification is brittle for subtle deception. Acceptable for cartoon characters who narrate their schemes aloud.
**Sources:** AgentTrustProvider SPI, #42 relationship memory wiring
**Exploration:** quick
**Status:** captured

## D9: Disposition signal mapping

**Choice:** Outcome-based mapping — map action type + result to BehavioralSignal and DispositionAxis. Only meaningful actions (STEAL, GIVE, PULL_ASIDE, USE, INTERACT) generate signals. MOVE, LOOK, WAIT are skipped to avoid noise.
**Alternatives:**
- Action-type only — static map regardless of outcome. Can't distinguish between attempting a steal and succeeding at one. Generates noise from failed mundane actions.
**Rationale:** Follows the same pattern as `importanceForAction()` already in the orchestrator. Avoids polluting disposition signals with noise from trivial actions. Deterministic, no LLM cost.
**Trade-offs:** Can't capture contextual nuance (stealing from ally vs enemy generates the same signal). Acceptable — disposition axes measure behavioral tendencies, not moral judgment.
**Depends on:** D8 (trust computation — disposition signals are a separate concern from trust scoring)
**Sources:** BehavioralSignalStore, DispositionSignalStore, importanceForAction() pattern
**Exploration:** quick
**Status:** captured
