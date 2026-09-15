## D1: Cognitive context source

**Choice:** Reuse tick queries — ManorNormFilter receives entity display names already resolved by the tick loop's CognitiveProfile queries
**Alternatives:**
- Independent query path — ManorNormFilter queries CognitiveProfile directly; adds per-tick query cost and a second cognitive query path
- Hybrid (tick queries + lightweight mindmap scan) — catches cognitively salient but physically absent entities; adds complexity for a marginal gain
**Rationale:** Zero additional queries. Norm relevance is driven by whatever the character is already attending to. The tick loop already computes EntityKnowledge for nearby entities — reusing that output avoids duplication.
**Trade-offs:** Norms can only activate based on physically nearby entities (those queried by the tick). A character thinking about an absent rival won't get norm activation for that rival. Acceptable because the tick's attention scope defines the character's actionable context.
**Sources:** CharacterCognition.java:94-128, ScenarioOrchestrator.java:243-257, #54 design spec §Per-Tick Query Flow
**Exploration:** quick
**Status:** captured

## D2: Relevance boost model

**Choice:** Binary presence — if the character has ANY cognitive context about an entity mentioned in the norm, the norm gets the full relevance boost
**Alternatives:**
- Scaled by engagement — boost proportional to belief/memory count; richer signal but needs tuning constants and couples norm scoring to belief counting
- Scaled by salience — boost weighted by TemporalFocus salience; most emergent but tightly couples norm scoring to TemporalFocus internals
**Rationale:** Simple, testable, avoids tuning. Cognitive depth already affects belief/trust sections — norms don't need to duplicate that signal. The budget already adapts to the situation via CognitiveBudget; adding engagement scaling on top risks over-engineering.
**Trade-offs:** A character deeply engaged with Penelope gets the same norm boost as one with a single belief. Acceptable because the norm's role is behavioral guidance, not cognitive depth — binary activation is semantically correct for "should this rule apply."
**Sources:** CognitiveBudget.java:3-22
**Exploration:** quick
**Status:** captured

## D3: Matching substrate

**Choice:** Case-insensitive word match — tokenize entity display names into words, check if any word (length >= 3) appears in norm text
**Alternatives:**
- Full display name substring — more precise but fails for partial name references ("Penelope" alone wouldn't match full "Penelope Pitstop")
- First-name + last-name individual match — same as word match but explicitly named-entity oriented; no practical difference
**Rationale:** Simple, no dependencies, good enough for natural-language norms. False positives rare with the word-length floor. The matching substrate is an implementation detail — the architecture (score by cognitive context overlap) is what matters. Can be swapped for embeddings later without changing the integration.
**Trade-offs:** Very short entity names (< 3 chars) won't match. Homonyms could cause false positives. Both are acceptable at current scale and addressable by swapping the substrate.
**Sources:** social-config.yaml (actual norm texts), GE-20260914-e3cb03 (profile data over prose directives)
**Exploration:** quick
**Status:** captured

## D4: Scoring + budget integration point

**Choice:** Scoring and budget gating in ManorNormFilter — it becomes a scoring/ranking utility that returns budget-gated results
**Alternatives:**
- Scoring in ManorContextStrategy, ManorNormFilter stays a pure filter — spreads logic across two classes, ManorNormFilter becomes a thin wrapper
- Norms as TemporalEntries scored by TemporalFocus — elegant unification but overloads TemporalFocus (designed for timestamped items with affect trajectories; norms are static)
**Rationale:** ManorNormFilter is the single-purpose utility for norm selection. Scoring is a natural evolution of filtering. CognitiveBudget already has maxNorms. ManorContextStrategy just wires the inputs — clean separation.
**Trade-offs:** ManorNormFilter takes on scoring responsibility beyond simple filtering. Acceptable because "score and select top N" is what filtering means in an attention-budget system.
**Sources:** ManorNormFilter.java:6-18, ManorContextStrategy.java:12-16, CognitiveBudget.java:3-22
**Exploration:** quick
**Status:** captured
