# D7 Mediator Synthesis — Unified vs Separate Storage

## Verdict: Unified Wins

Unified mindmap (Tier 3) with typed Thing trait facades. Separate in-process tiers (1+2) for hot-path state.

## Key Arguments Assessed

**Cross-concept inference (unified):** Correct. "Why does HC distrust Penelope?" spans trust edges, belief nodes, episodic events. One graph traversal vs three store lookups with manual correlation.

**Hot-path performance (separate):** Correct but addressed. Three-tier model means trust updates are Tier 2 (in-process), not Tier 3. The 1,500 trust events never hit the graph during gameplay.

**Soar/ACT-R (separate):** The strongest argument — but misread. Soar has functionally separate modules but structurally unified working memory. ACT-R has separate buffers feeding a shared production system. Both architectures are functionally modular, structurally unified at the reasoning layer. The three-tier model mirrors this.

**IoT (separate):** Valid but out of scope. Raw sensor data isn't cognitive state. Anomaly conclusions belong in graph; telemetry stays in time-series stores.

**Session-scoped data (separate):** Valid but already handled. Negotiation state is Tier 1/2, not Tier 3.

**Debuggability (separate):** Genuine concern. Mitigated by Thing trait interfaces — `node.as(Belieflike.class)` gives the same API surface as a dedicated store.

## Incorporation from Losing Position

Each cognitive type needs its own typed API — application code should call `beliefs.forAgent(id)`, not write graph queries. The Thing trait system already provides this. No separate facade classes needed.

## Sources

- AriGraph (IJCAI-25) — unified semantic+episodic outperforms separate
- Graphiti/Zep — temporally-aware KGs outperform MemGPT on deep retrieval
- Soar architecture — functionally modular, structurally unified
- Memanto (2026) — argues against graph complexity (acknowledged but countered by three-tier model)
- Neocortex CognitiveProfile, PerspectivalResolver, ConversationBridge — existing infrastructure built for unified storage
