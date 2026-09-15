# Decisions — Issue #56: Move SocialConfig to YAML

## D1: Config source — local YAML file instead of Eidos extensionData

**Choice:** Create a standalone `social-config.yaml` in `META-INF/eidos/` (alongside descriptor YAML files). Parse with a static `ManorSocialConfigLoader.load()` utility. The original issue assumed Eidos `AgentDescriptor` had an `extensionData` field — it does not. A local YAML file achieves the same goal (config in YAML, not hardcoded Java) without requiring an upstream Eidos API change.
**Alternatives:**
- Add extensionData to Eidos upstream — correct long-term but adds cross-repo dependency, increases scope from S to M
- Defer — hardcoded config works, but leaves the config locked in Java code
**Rationale:** The goal is data/code separation — character personality config should be editable without recompiling. A local YAML file is the simplest path. File an upstream Eidos issue for extensionData as a follow-up; when it lands, the YAML content can migrate into descriptors.
**Trade-offs:** Config split across two file types (descriptors-*.yaml for capabilities/disposition, social-config.yaml for drives/norms/beliefs). Acceptable: the files live in the same directory and the split is temporary.
**Sources:** AgentDescriptor.java (no extensionData field), descriptors-baseline.yaml, blog 2026-09-13
**Exploration:** quick
**Status:** captured

## D2: Loading mechanism — static loader alongside descriptors

**Choice:** Static utility `ManorSocialConfigLoader.load()` reads `social-config.yaml` from the classpath via `Thread.currentThread().getContextClassLoader().getResourceAsStream()`. Returns `Map<String, SocialConfig>`. Called once during ScenarioOrchestrator initialization. SocialConfig.forCharacter() is replaced by a direct map lookup — no fallback to hardcoded data (hardcoded map removed entirely).
**Alternatives:**
- CDI bean loader — more Quarkus-idiomatic but adds CDI wiring for a simple config read
- Quarkus @ConfigMapping — most native but requires YAML under application.yaml, not a standalone file
**Rationale:** The descriptor YAML files are already loaded via classpath resource streams (by EidosLoader). Following the same pattern keeps the loading mechanism consistent. No CDI needed for a static config file read.
**Trade-offs:** Not hot-reloadable (requires restart to pick up changes). Acceptable: character config doesn't change at runtime.
**Sources:** ScenarioOrchestrator.java:158 (current SocialConfig.forCharacter call site)
**Exploration:** quick
**Status:** captured
