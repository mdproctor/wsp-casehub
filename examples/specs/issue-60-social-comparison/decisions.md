## D1: Budget gate for compare()

**Choice:** Gate by drives — only call compare() when the character has scheming or suspicion drives (intensity > 0.5)
**Alternatives:**
- Unconditional for all characters — simpler but adds prompt noise for characters who don't think socially
- Gate by budget flag — adds a boolean to CognitiveBudget; flexible but over-engineers the existing record
**Rationale:** Theory-of-mind is a cognitive effort characters don't always invest. Matches #54 spec intent. Keeps prompt compact for simple characters.
**Trade-offs:** Characters without scheming/suspicion drives never get social awareness. Acceptable — those characters don't need it narratively.
**Sources:** #54 design spec §Per-Tick Query Flow step 3, SocialConfig drives in social-config.yaml
**Exploration:** quick
**Status:** captured

## D2: Section content — perception-level translation

**Choice:** Translate PAD divergence into character-perspective perception statements. Neither generic prose nor raw metrics.
**Alternatives:**
- Minimal contrast lines ("you sense tension") — too vague, LLM has no signal about what kind of tension
- PAD-annotated lines ("divergence 0.7") — leaks mechanism, requires domain knowledge to interpret
**Rationale:** Perception-level statements are data (not directives), self-contained (no PAD theory needed), and specific enough to act on. Follows GE-20260914-e3cb03 principle extended by GE-20260915-2b5e80 for comparative state.
**Trade-offs:** Requires a translation method (~5-10 template patterns). Acceptable for S/Med scope.
**Sources:** GE-20260914-e3cb03 (profile data over directives), GE-20260915-2b5e80 (comparative data translation)
**Exploration:** deep-analysis
**Status:** captured

## D3: Compare target entity

**Choice:** Compare on each nearby character — query compare(X_node, {self, X}) for each nearby character X
**Alternatives:**
- Compare on self — how do nearby characters see ME; useful but less actionable for scheming
- Bidirectional (both) — most complete but doubles query cost, noisy sections
**Rationale:** Gives the acting character insight into whether their perception of someone matches that person's self-image. Natural for social scheming ("I see Peter as a fool, but he sees himself as a hero — I can exploit that gap").
**Trade-offs:** Doesn't show how others see the acting character. Acceptable — compare on self can be added later if needed.
**Sources:** CognitiveProfile.compare() API, SocialComparison.compare() API, PerspectivalComparison record
**Exploration:** quick
**Status:** captured
