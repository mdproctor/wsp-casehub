# Decisions — Phase A: Social Cognition Layer

## D1: Integration depth — Wire existing platform (revised from Hybrid)

**Choice:** Wire existing neocortex/blocks cognitive stack into wacky-manor. Beliefs, trust, drives, norms, principles all backed by neocortex mindmap (Tier 3) via CognitiveProfile + Thing trait projections. Tier 1 (working memory) and Tier 2 (episodic buffer) remain in-process.
**Alternatives:**
- Additive — keep all existing systems, only add new layers. Lower risk but misses the opportunity to showcase blocks trust/memory subsystems.
- Substitutive — replace ManorGoal*, ManorReflection*, ManorTrust*, ManorDisposition* with blocks equivalents. Better platform showcase but rewires the working cognitive stack; high risk.
**Rationale:** Trust and disposition are simple subsystems with clean replacement targets in blocks. Goal/plan/reflection are complex, working, and not worth the risk of replacement. Beliefs, drives, and norms are genuinely new capabilities — layering them preserves the working stack.
**Trade-offs:** Two integration seams — existing goal/plan/reflection system and new blocks social cognition exist side by side. More code than a pure substitutive approach.
**Sources:** ScenarioOrchestrator.java, ManorTrustProvider.java, ManorDispositionRecorder.java, blocks.trust, blocks.memory, blocks.agentic.social, blocks.normative
**Exploration:** quick
**Status:** captured

## D2: Configuration source — Eidos descriptors + CognitiveDerivationEngine

**Choice:** Social cognition defaults derived from personality via CognitiveDerivationEngine (trust formation rate, conflict interpretation, curiosity config from Jungian function stack). Per-character overrides in Eidos extensionData.
**Alternatives:**
- Separate YAML config — new files per character. Keeps Eidos clean but splits character definition across two files.
- Java code — hard-coded per-character social config. Simple but defeats the purpose of demonstrating declarative configuration.
**Rationale:** Eidos already owns personality traits, goals, and capabilities. Social cognition (drives, norms, beliefs) is a natural extension of personality. One file per character, one source of truth.
**Trade-offs:** Eidos descriptor schema must be extended, which may require eidos-api changes. If the schema extension is complex, this could spill into the eidos repo.
**Sources:** Eidos YAML descriptors in wacky-manor/src/main/resources/eidos/
**Exploration:** quick
**Status:** captured

## D3: Influence mode — Prompt-visible

**Choice:** Inject beliefs, drives, and norms directly into the observation text the LLM sees. The LLM reasons about them explicitly.
**Alternatives:**
- Pre-LLM filtering — influence memory recall weighting and observation filtering but don't appear in prompts. More subtle, harder to demo.
- Both — drives/norms in prompts, beliefs also influence pre-LLM memory weighting. More powerful but more complex integration.
**Rationale:** Most transparent approach. The LLM can reason about "You believe X" and "Your drive to scheme is strong" explicitly. Best for demo/showcase — observers can see exactly what cognitive inputs shaped the character's decision.
**Trade-offs:** Prompt space — beliefs/drives/norms add to observation length, reducing space for world state and memories. Need to keep sections concise.
**Sources:** ObservationBuilder.java, CognitiveObservationSections (blocks)
**Exploration:** quick
**Status:** captured

## D4: Phasing — Separate branches

**Choice:** Phase A (social cognition) and Phase B (mindmap memory) as separate issues/branches, landed independently.
**Alternatives:**
- Combined — one branch, both phases. Faster if both go smoothly but a blocker in either delays everything.
**Rationale:** Each phase is independently valuable and testable. Social cognition can land and be validated before mindmap migration begins. Reduces blast radius.
**Trade-offs:** Two branch cycles instead of one. Phase B may need to adapt to Phase A's integration points, but this is minor.
**Sources:** POC-SPEC.md phase structure
**Exploration:** quick
**Status:** captured

## D5: Memory architecture — Three-tier with mindmap-native cognition

**Choice:** Three-tier memory model. Tier 1 (working memory, in-process, per-tick). Tier 2 (episodic buffer, in-process, short-lived). Tier 3 (knowledge graph, neocortex mindmap, write-on-consolidation-only). Consolidation bridges Tier 2 → 3 during sleep game mechanic.
**Alternatives:**
- Flat memory only — current approach. No structured knowledge, no consolidation.
- Mindmap for everything — writes to graph every tick. Too expensive, wrong model for ephemeral state.
**Rationale:** Neocortex mindmap is designed for consolidated knowledge, not high-frequency state. Three tiers mirror human cognition: working memory → hippocampal buffer → neocortical long-term storage.
**Trade-offs:** More complex architecture. Consolidation logic must decide what graduates from Tier 2 → 3 vs gets pruned.
**Sources:** Neocortex ConsolidationScheduler, AgentExperienceService (current Tier 2), AriGraph (IJCAI-25), Graphiti/Zep research
**Exploration:** deep-analysis
**Status:** captured

## D6: Norms vs constraints — Principles + contextual norms

**Choice:** Eidos constraints stay in system prompt (identity: "you ARE this way"). Principles copied to observation (active guidance). Norms are situation-specific behavioral guidance filtered by context — a different concept from identity-level constraints.
**Alternatives:**
- Enhance constraints with priority/tags — one system for all behavioral rules. But constraints are in system prompt; dynamic rules need to be in observation.
- Duplicate — constraints and norms as separate parallel systems. Architectural debt.
**Rationale:** System prompt = identity (stable). Observation = active cognition (dynamic). Constraints reinforce identity; principles bridge to active guidance; norms are context-filtered behavioral rules. Three distinct semantic roles.
**Trade-offs:** Three levels of behavioral guidance (constraints, principles, norms) may confuse. Mitigation: clear naming and distinct rendering sections.
**Sources:** Eidos descriptors-composite.yaml (existing constraints), design review R1-07
**Exploration:** deep-analysis
**Status:** captured

## D7: Storage model — Unified mindmap with Thing trait facades

**Choice:** All consolidated cognitive state lives on neocortex mindmap (Tier 3). Per-concept access via Thing trait projections: `node.as(Belieflike.class)`, `node.as(Intentional.class)`. Collection queries via CognitiveProfile filtered by node type. No separate BeliefStore, TrustStore, etc. — the Thing API IS the facade.
**Alternatives:**
- Separate dedicated stores (BeliefStore, TrustStore, etc.) — O(1) lookups, simpler API, but fragments cross-concept queries and ignores existing neocortex infrastructure (CognitiveProfile, PerspectivalResolver, ConversationBridge).
- Separate stores with graph sync — worst of both worlds, duplicated state.
**Rationale:** Multi-agent debate (3 agents). Unified won on merit. Soar/ACT-R research, properly read, supports unified long-term storage with separate working buffers — which is the three-tier model. Cross-concept inference ("why does HC distrust Penelope?") is a single graph traversal. Existing neocortex infrastructure was built for unified storage. Thing trait interfaces give typed per-concept APIs without separate wrapper classes.
**Trade-offs:** Steeper on-ramp. Graph query debugging harder than HashMap. Mitigation: typed facades via Thing traits give clean API surface; graph is an implementation detail.
**Depends on:** D5 (three-tier model — Tier 3 is the unified store)
**Sources:** Soar/ACT-R architectures, AriGraph (IJCAI-25), Graphiti/Zep, Memanto, neocortex Thing API, CognitiveProfile, PerspectivalResolver
**Exploration:** multi-agent-debate
**Status:** captured
