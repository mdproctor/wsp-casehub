# Decisions — Phase A: Social Cognition Layer

## D1: Integration depth — Hybrid

**Choice:** Hybrid — replace trust + disposition (simple, low-risk swaps), layer beliefs + social drives + norms additively as new cognitive inputs.
**Alternatives:**
- Additive — keep all existing systems, only add new layers. Lower risk but misses the opportunity to showcase blocks trust/memory subsystems.
- Substitutive — replace ManorGoal*, ManorReflection*, ManorTrust*, ManorDisposition* with blocks equivalents. Better platform showcase but rewires the working cognitive stack; high risk.
**Rationale:** Trust and disposition are simple subsystems with clean replacement targets in blocks. Goal/plan/reflection are complex, working, and not worth the risk of replacement. Beliefs, drives, and norms are genuinely new capabilities — layering them preserves the working stack.
**Trade-offs:** Two integration seams — existing goal/plan/reflection system and new blocks social cognition exist side by side. More code than a pure substitutive approach.
**Sources:** ScenarioOrchestrator.java, ManorTrustProvider.java, ManorDispositionRecorder.java, blocks.trust, blocks.memory, blocks.agentic.social, blocks.normative
**Exploration:** quick
**Status:** captured

## D2: Configuration source — Eidos descriptors

**Choice:** Extend character Eidos YAML descriptors with new `social:` sections for drives, norms, and initial-beliefs.
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
