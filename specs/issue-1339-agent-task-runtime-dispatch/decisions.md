## D1: Dispatch handler placement

**Choice:** Inline private method in `CaseContextChangedEventHandler`
**Alternatives:**
- Separate `AgentTaskDispatcher` service — more testable in isolation but adds indirection and breaks the pattern where every target type dispatches from the same handler
**Rationale:** All other target types (CapabilityTarget, JudgmentTarget, SignalTarget, SubCaseTarget) dispatch from the same `publishByTarget()` switch in `CaseContextChangedEventHandler`. Adding a new private method follows the established pattern.
**Trade-offs:** The handler class is already large (~1000 lines). One more private method adds to that, but consistency is more valuable than size reduction here.
**Sources:** `CaseContextChangedEventHandler.publishByTarget()` (line 503), `publishJudgmentSchedule()` (line 846)
**Exploration:** quick
**Status:** captured

## D2: Agent match selection strategy

**Choice:** First-match (hardcoded)
**Alternatives:**
- Wire through `StrategyResolver` / `AgentRoutingStrategy` SPI — more extensible but overengineered for the initial dispatch path; agent capability matching is a different routing problem than worker routing
**Rationale:** `AgentRegistry.find()` already returns matches ordered by quality. The issue specifies "default strategy: first result". `AgentWorkerFunctionHandler.executeDeferred()` uses the same pattern.
**Trade-offs:** No pluggable selection strategy. Acceptable for now — strategy can be added later via the same SPI extension point.
**Sources:** Issue #1339 ("Select best match from returned List<AgentMatch> (default strategy: first result)"), `AgentWorkerFunctionHandler.executeDeferred()` (line 112-118)
**Exploration:** quick
**Status:** captured

## D3: Dispatch-to-execution architecture (hybrid)

**Choice:** Dispatch resolves capability→agentId (selection); execution resolves agentId→descriptor→invoker (mechanics). New `AgentWorkerFunction.ofAgent(AgentTaskConfig)` construction path.
**Alternatives:**
- Resolve capability→slot at dispatch, defer to `ofSlot` at execution — two lookups on different keys; slot could match a different agent if registrations change between dispatch and execution
- Resolve fully at dispatch, use `eager(invoker)` — single lookup but puts prompt rendering and invoker construction in the dispatch handler (wrong separation of concerns)
**Rationale:** Each handler does exactly its job: dispatch makes the selection decision (which agent?), execution handles the mechanics (how to invoke it?). The `agentId` is the stable identity of the selected agent. `findById(agentId, tenancyId)` at execution time picks up the latest descriptor without key mismatch risk.
**Trade-offs:** Requires a new `AgentTaskConfig` record and a third construction mode on `AgentWorkerFunction`. Small API surface cost for clean separation.
**Sources:** `AgentWorkerFunctionHandler.executeDeferred()` (line 108-147), `AgentRegistry.findById()`, `CaseContextChangedEventHandler.publishByTarget()` (line 503)
**Exploration:** deep-analysis
**Status:** captured
