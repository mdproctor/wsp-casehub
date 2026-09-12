# Design Journal — issue-52-social-cognition-layer

## 2026-09-11/12 — Design session: social cognition for wacky-manor

**Outcome:** Comprehensive design for wiring neocortex/blocks cognitive stack into wacky-manor. Started as "add beliefs and drives" → evolved into a mindmap-native cognitive architecture with three-tier memory and consolidation game mechanic.

**Key evolution:**
1. Started with separate stores (BeliefStore, TrustStore) → audit found blocks trust is vouch-oriented, not suitable
2. Discovered neocortex has a complete cognitive stack already built (CognitiveProfile, PerspectivalResolver, CognitiveDerivationEngine, ConversationBridge) — missed on first audit
3. Multi-agent debate: unified mindmap vs separate stores → unified wins, Thing trait projections replace separate store classes
4. Three-tier memory: working (per-tick) → episodic buffer (in-process) → knowledge graph (write-on-consolidation)

**Decisions captured:** D1-D7 in decisions.md. D7 validated via multi-agent debate.

**Upstream issues filed:**
- neocortex#322 — cognitive node types + trait interfaces (S)
- neocortex#323 — consolidateNow(tenantId) trigger (XS)
- blocks#260 — cognitive observation section renderers (S)

**Also created:** examples#51 — YAML tutorials using Pages scenario engine (needs slot)

**Implementation plan:** 7 tasks in 4 batches — Batch 1-2 (refactoring, no upstream deps) can start immediately
