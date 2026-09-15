## D1: Test weight

**Choice:** Single-turn + inline LLM judge — construct system prompt + observation with cognitive sections, invoke LLM once per scenario, judge with a structured prompt
**Alternatives:**
- Full scenario simulation — multi-turn autonomous runs; realistic but heavyweight (~minutes/test) and hard to attribute behavior to specific cognitive inputs
- A/B comparison — run with and without cognitive sections; strongest signal but doubles cost per test
**Rationale:** Fast (~5 sec/test), focused, isolates cognitive influence to specific observation sections. Each test is a controlled experiment: known cognitive state → scenario → response → judge verdict.
**Trade-offs:** Single-turn doesn't capture multi-turn accumulation effects. Acceptable because the tests verify influence, not realism — multi-turn effects are tested by BaselineLayerTest.
**Sources:** PromptQualityTest.java (existing eval pattern), BaselineLayerTest.java (existing scenario pattern), GE-20260804-2cd3da (split-model eval)
**Exploration:** quick
**Status:** captured

## D2: Judge approach

**Choice:** Inline LLM judge prompt — use AgentProvider directly with a structured judge prompt ("Given this cognitive state, does this response reflect [trait]? Score 0-5."), parse JSON response. No new judge class.
**Alternatives:**
- New CognitiveInfluenceJudge class — reusable, follows existing judge pattern, but over-engineered for 4 tests
- Keyword + heuristic checking — fast but brittle, misses paraphrasing
**Rationale:** Keeps everything in the test class, matches S/Med scope. The judge prompt is the test — it specifies what cognitive influence to look for. If this grows to 10+ tests, promote to a dedicated judge class later.
**Trade-offs:** No reusability across test classes. No typed result record. Acceptable for 4 tests.
**Sources:** EvalJudgeProducer.java (existing judge CDI pattern), FunctionActivationJudge (existing judge structure)
**Exploration:** quick
**Status:** captured

## D3: Test structure

**Choice:** Single test class with parameterized scenarios via @MethodSource — CognitiveScenario records define (name, agentId, userPrompt, judgeCriteria)
**Alternatives:**
- Separate test methods per scenario — simpler to read but repetitive boilerplate
- Separate test classes per cognitive element — over-engineered for 4 tests
**Rationale:** DRY, extensible — adding a scenario is one record. Matches existing pattern in PromptQualityTest which iterates over profiles and characters.
**Trade-offs:** Parameterized tests are harder to debug individually. Mitigated by clear scenario names in test output.
**Sources:** PromptQualityTest.java (iteration pattern), social-config.yaml (actual character data)
**Exploration:** quick
**Status:** captured
