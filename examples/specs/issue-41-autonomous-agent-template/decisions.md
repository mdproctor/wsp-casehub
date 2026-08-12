# Decisions — #43 Goal Lifecycle

## D1: Goal formation timing

**Choice:** Post-reflection async — goal formation chains after reflection on the same virtual thread. Goals appear on the character's next tick.
**Alternatives:**
- End-of-tick sync — adds tick latency, awkward with async reflection (race between reflection completion and tick end)
- Separate cadence — independent timer/interval, adds complexity for no clear benefit
**Rationale:** Reflection already runs on a virtual thread when importance thresholds trigger. Goal formation is the natural next step in that chain: ingest → reflection trigger → synthesize insights → form goals. No tick loop latency, no race conditions. One-tick delay before goals appear models cognitive latency.
**Trade-offs:** The full chain (reflection + formation + revision) is 3 LLM calls (6-30 seconds). Multiple ticks may elapse before goals appear — but `currentPlan` handles reactive intent per-tick while goals handle strategic direction post-reflection. This maps to System 1 (plan, reactive, per-tick) vs System 2 (goals, deliberative, post-reflection).
**Exploration:** quick
**Status:** captured
**Review notes:** R1-04 identified that latency was understated (3 LLM calls, not 1). R1-01 identified that removing per-tick newGoals/dropGoals eliminates reactive goal agency — addressed by the plan/goal cognitive split: currentPlan is the reactive mechanism, goals are the strategic layer.

## D2: CaseDefinition on engine SPIs

**Choice:** Challenge the SPI — remove CaseDefinition from GoalFormationContext and GoalRevisionContext entirely.
**Alternatives:**
- Pass null — works but null parameters are a code smell, and nullable fields that no implementation reads are dead weight
- Thin manor wrapper — construct a minimal CaseDefinition, but it would be pretending the manor has cases
**Rationale:** Neither LlmGoalFormationStrategy nor LlmGoalRevisionStrategy reads context.definition(). The evaluators extract all relevant data (agentId, goals, insights, memories) into the context's own fields before constructing it. CaseDefinition on an SPI interface couples every implementor to a ~30-field engine-internal model. If a strategy needs case context in the future, it can inject CaseDefinitionRegistry via CDI.
**Trade-offs:** A future strategy that genuinely needs case context must inject it rather than receiving it as a parameter. CDI injection is the standard pattern for this.
**Exploration:** quick
**Status:** captured (implemented as casehubio/engine#897)

## D3: Goal storage model

**Choice:** AgentRegistry (engine pattern) — new goals registered on the AgentDescriptor via AgentRegistry.register(). Single source of truth for all goals.
**Alternatives:**
- CharacterState (local) — keep goals on CharacterState using AgentGoal type instead of DynamicGoal. Simpler but diverges from engine pattern.
- Both (layered) — identity goals from AgentRegistry, situational goals on CharacterState. Preserves two-tier model but adds complexity.
**Rationale:** The engine's GoalFormationEvaluator already uses this pattern. ObservationBuilder already calls resolveGoals() which reads from AgentRegistry. Single storage eliminates the identity/situational split and removes DynamicGoal, dynamicGoals collection, newGoals/dropGoals from AgentResponse, and goal processing from the tick loop. The currentPlan mechanism handles reactive per-tick intent that DynamicGoal previously served.
**Trade-offs:** AgentRegistry writes during the game — minor overhead per goal formation. Descriptor immutability means building a new descriptor with updated goals (toBuilder().goals(merged).build()) on each formation. Concurrent writes need cooldown protection (R1-08).
**Exploration:** quick
**Status:** captured

## D4: Goal outcome tracking

**Choice:** LLM-assessed via reflection — when reflection runs, the synthesizer also assesses goal progress (advancing, stalled, failed), producing structured GoalOutcomeCounts alongside insights.
**Alternatives:**
- Mechanical action mapping — map actions to goals mechanically. Precise but brittle, requires hand-coded mappings per goal.
- Defer revision entirely — implement only GoalFormationStrategy, skip revision. Gets core value faster but goals accumulate without cleanup.
**Rationale:** The manor's design principle is "LLM-driven autonomy by default, mechanical support only where the LLM consistently fails." Goal outcome assessment is a judgment call (is this goal progressing?) that the LLM handles naturally from the agent's memory context. Consistent with how reflection already works. The synthesizer produces GoalOutcomeCounts per goal to populate GoalRevisionContext.counts — not left empty.
**Trade-offs:** Non-deterministic — the LLM might assess outcomes inconsistently across reflections. Acceptable because goal revision is advisory (the evaluator validates before applying).
**Depends on:** D1 (goal formation timing — revision runs in the same async chain)
**Exploration:** quick
**Status:** captured
**Review notes:** R1-03 identified that the existing GoalOutcomeCounts infrastructure was unaddressed. Resolved: the reflection synthesizer produces structured counts, not just free-text insights.

## D5: Goal lifecycle transitions

**Choice:** GoalRevisionStrategy handles all lifecycle transitions — revision, abandonment, and completion — via a GoalRevisionAction enum (REVISE, ABANDON, COMPLETE) on RevisedGoal.
**Alternatives:**
- Threshold-based auto-drop — after N failures, automatically remove. Deterministic but doesn't account for context.
- Formation handles it — expand GoalFormationProposal to include goals to drop. Mixes formation and cleanup responsibilities.
- Separate mechanisms for each transition — more components, harder to reason about.
**Rationale:** Single mechanism for all post-formation lifecycle transitions. The LLM assesses whether a goal should be revised, abandoned, or marked complete. The action enum is type-safe and explicit — no string conventions or semantic overloading.
**Trade-offs:** Requires engine API change to GoalRevisionProposal.RevisedGoal (casehubio/engine#903).
**Depends on:** D4 (outcome tracking provides the data for lifecycle decisions)
**Exploration:** quick
**Status:** revised (originally D5 + D7, merged after decision review)
**Review notes:** R1-02 identified D7's string convention as a workaround. R1-05 identified that goal completion was unmodeled. R1-07 identified ambiguous RevisedGoal semantics. All three resolved by the GoalRevisionAction enum upstream (engine#903).

## D6: Overall architecture

**Choice:** Full engine SPI implementation — add casehub-engine-api dependency, implement GoalFormationStrategy and GoalRevisionStrategy as manor CDI beans, create ManorGoalEvaluator to orchestrate the flow.
**Alternatives:**
- Local strategy pattern — define equivalent interfaces in the manor, no engine dependency. Duplicates the SPI contract and can't swap in engine strategies.
- Formation only — implement GoalFormationStrategy, defer revision. Smaller scope but no goal cleanup, splits the issue.
**Rationale:** The #41 epic goal is "wire full neocortex + engine cognitive stack." The engine-api dependency is lightweight (SPI interfaces and context records, no runtime). Both formation and revision give the full lifecycle: goals emerge from reflection, evolve from outcomes, get abandoned or completed when appropriate.
**Trade-offs:** New dependency on casehub-engine-api. More components than the current simple model (strategy + evaluator vs. inline processing).
**Exploration:** quick
**Status:** captured

## D7: Priority evolution — deferred

**Choice:** Goal priority is fixed at formation for #43. Priority evolution deferred to a future issue.
**Rationale:** GoalRevisionProposal.RevisedGoal does not currently have a revisedPriority field. Adding one is a valid extension but out of scope for #43 — the GoalRevisionAction enum (engine#903) is the priority change. Priority evolution can be added to RevisedGoal in a follow-up.
**Review notes:** R1-06 identified that priority cannot evolve after formation. Acknowledged as a limitation, deferred.
**Status:** captured
