## D1: Output contract

**Choice:** Initiation decision + content
**Alternatives:**
- Raw thoughts only — returns List<InnerThought> with motivation scores. Maximum flexibility but pushes dispatch complexity to every consumer.
- Full message dispatch — InnerLife posts directly via MessageDispatcher. Tighter coupling to qhorus, removes consumer control over dispatch timing and channel selection.
**Rationale:** tick() returns a sealed type: Silent (no motivation), Reflecting (thinking but not ready), Initiated(content, channelHint, motivationScore). The consuming app decides how to dispatch — post to channel, queue for human review, or discard. This matches PersonalityEvolution's EvolutionTick pattern and blocks' role as a library (not a framework). The sealed type gives consumers exhaustive pattern matching.
**Trade-offs:** Consumers must wire dispatch logic. Accepted because dispatch is domain-specific (Discord bot posts differently from an email agent).
**Sources:** PersonalityEvolutionOrchestrator.EvolutionTick (blocks, #118), SummarisationRunner.tick() (blocks), Liu et al. CHI 2025 (proactive agents with inner thoughts)
**Exploration:** quick
**Status:** captured

## D2: Motivation model

**Choice:** Hybrid: heuristic gate + LLM score
**Alternatives:**
- LLM-scored only — always asks LLM to score motivation. Natural but wastes calls during quiet periods.
- Heuristic only — scores on observation count, time since last spoke, novelty. No LLM cost but less context-aware.
**Rationale:** Two-phase evaluation. Phase 1 (heuristic): checks minimum observations since last initiation, minimum time gap, and novelty score (D3). If any heuristic fails → return Silent immediately (zero LLM cost). Phase 2 (LLM): asks AgentProvider to score motivation 0.0–1.0 against current context + affordances + reflections. Above configurable threshold → Initiated. This gives the best cost/quality trade-off — quiet periods never invoke the LLM, active periods get nuanced motivation assessment.
**Trade-offs:** Two-phase adds complexity. The heuristic gate may filter out genuinely motivated responses during quiet periods. Accepted because the heuristic thresholds are configurable and erring toward silence is safer than over-posting.
**Sources:** Deng et al. AAAI 2025 (human-centered proactive agents: intelligence, adaptivity, civility), Liu et al. CHI 2025 (motivation threshold)
**Exploration:** quick
**Depends on:** D1 (output contract determines what motivation evaluation produces)
**Status:** captured

## D3: System 1/2 fast path

**Choice:** Novelty-gated reflection
**Alternatives:**
- Configurable skip rules — consumer-defined ActivationRule<ObservationResult> before reflection. More flexible but requires per-consumer configuration.
- Always reflect — LLM motivation score naturally handles staleness. Simpler but wastes LLM calls on quiet periods.
**Rationale:** Before calling ReflectionOrchestrator (which invokes the LLM), compute novelty of accumulated observations against recent history using token-level Jaccard distance. Low novelty (below configurable threshold) → skip reflection, return Silent. High novelty → proceed to full reflection + motivation evaluation. This is the System 1 (fast, cheap) gate before System 2 (slow, expensive) processing. The novelty scorer is a pure function on observation text — no LLM call, no external dependency.
**Trade-offs:** Token-level Jaccard is crude — semantically novel content with similar vocabulary may be filtered. Accepted because false negatives (skipping a motivated initiation) are low-cost (the next tick will catch it if observations continue accumulating), while false positives (unnecessary LLM calls) have real cost.
**Sources:** Research doc §2.8 (System 1/System 2 fast path fold-in), DPT-Agent 2025 (Dual Process Theory with ToM)
**Exploration:** quick
**Depends on:** D2 (fast path is Phase 0 before the heuristic gate)
**Status:** captured

## D4: Civility constraints

**Choice:** Built-in civility config
**Alternatives:**
- Delegate to Watchdog — create AGENT_OVER_POSTING condition type. Requires qhorus SPI extension; Watchdog conditions are channel-scoped, not agent-scoped.
- Composable CivilityConstraint SPI — @FunctionalInterface with check(). More extensible but more types for consumers to implement.
**Rationale:** InnerLife owns a CivilityGuard with three configurable constraints: minimum gap between initiations (default: 5 minutes), maximum initiations per time window (default: 3 per hour), cooldown after N consecutive initiations without response (default: 2 → cooldown until someone else posts). These are agent-level constraints, not channel-level — Watchdog operates on channels, not individual agents. The guard is checked as the first step of tick(), before novelty scoring or LLM calls. All three parameters are configurable per-agent via PersonalityEvolutionConfig-style preferences.
**Trade-offs:** Not extensible beyond the three built-in constraints. If a domain needs custom civility rules (e.g., "don't initiate during business hours"), they must wrap InnerLife rather than plug into it. Accepted because the three built-in constraints cover the core civility requirements identified in the literature (Liu et al., Deng et al.).
**Sources:** Liu et al. CHI 2025 (civility constraints), Deng et al. AAAI 2025 (adaptivity + civility), Watchdog (qhorus) — channel-scoped, not agent-scoped
**Exploration:** quick
**Depends on:** D1 (civility gate runs before output production)
**Status:** captured
