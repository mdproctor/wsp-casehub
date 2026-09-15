# Move SocialConfig to YAML — Design Spec

**Issue:** casehubio/examples#56
**Epic:** casehubio/examples#53 (Phase B: mindmap-native cognitive integration)
**Date:** 2026-09-15
**Status:** Draft

---

## Problem

SocialConfig uses a hardcoded Java map for 5 characters' drives, norms, and initial beliefs. Adding or modifying character personality config requires editing Java source and recompiling. The config should live in YAML alongside other character descriptor files.

---

## Architecture

### YAML Format

`META-INF/eidos/social-config.yaml` — one entry per character:

```yaml
hooded-claw:
  drives:
    - type: scheming
      intensity: 0.9
      description: "Compelled to hatch elaborate plans against Penelope"
    - type: self-preservation
      intensity: 0.7
      description: "Avoids direct confrontation, prefers subterfuge"
    - type: dominance
      intensity: 0.6
      description: "Must be the most powerful person in every room"
  norms:
    - rule: "Never help Penelope directly"
      priority: 10
    - rule: "Maintain a veneer of charm in public"
      priority: 5
    - rule: "Protect personal schemes from discovery"
      priority: 8
  initial-beliefs:
    - key: penelope-awareness
      value: "Penelope is naive and trusts too easily"
    - key: peter-threat
      value: "Peter Perfect is protective but predictable"

penelope-pitstop:
  # ... same structure
```

### ManorSocialConfigLoader

Static utility class:

```java
public final class ManorSocialConfigLoader {
    public static Map<String, SocialConfig> load() { ... }
    public static Map<String, SocialConfig> load(String resourcePath) { ... }
}
```

- Reads YAML from classpath via `getContextClassLoader().getResourceAsStream()`
- Parses with Jackson's `ObjectMapper` (YAML module already on classpath via Quarkus)
- Maps each top-level key to a `SocialConfig` record
- Handles `initial-beliefs` (kebab-case in YAML) → `initialBeliefs` (camelCase in Java)
- Returns unmodifiable map
- Throws `IllegalStateException` if the file is missing or unparseable (fail-fast at startup)

### SocialConfig Changes

- Remove the hardcoded `CONFIGS` map
- Remove `forCharacter(String agentId)` static lookup
- Remove `hasConfig(String agentId)` static lookup
- Keep the record definition and sub-records (Drive, NormEntry, InitialBelief) unchanged

### ScenarioOrchestrator Changes

In `runScenario()`, load the config map once before the character loop:

```java
var socialConfigs = ManorSocialConfigLoader.load();
```

Replace `SocialConfig.forCharacter(entry.getKey())` with:

```java
var socialCfg = socialConfigs.getOrDefault(entry.getKey(), SocialConfig.empty());
```

---

## What Needs Building

### New types

| Type | Package | Purpose |
|---|---|---|
| `ManorSocialConfigLoader` | `manor.agent` | Static YAML loader returning `Map<String, SocialConfig>` |

### New files

| File | Purpose |
|---|---|
| `META-INF/eidos/social-config.yaml` | Character social config data (drives, norms, beliefs) |

### Modified types

| Type | Change |
|---|---|
| `SocialConfig` | Remove hardcoded CONFIGS map, forCharacter(), hasConfig() |
| `ScenarioOrchestrator` | Load social config from YAML at startup, pass to character loop |

---

## Testing

**Unit tests:**
- ManorSocialConfigLoaderTest — load from test YAML, verify all 5 characters parsed correctly
- Drive intensity validation (0-1 range)
- Missing file → IllegalStateException
- Empty file → empty map
- Character not in YAML → SocialConfig.empty() via getOrDefault

**Integration verification:**
- Run existing SocialCognitionIntegrationTest — validates SocialConfig rendering unchanged
- Run existing CharacterCognitionTest — validates drives/norms/beliefs rendering
- All 450 tests pass unchanged

---

## What's NOT in Scope

- Eidos extensionData API change — upstream follow-up issue
- Adding social config for composite-only characters (Muttley, etc.) — separate issue
- Hot-reload of YAML config at runtime

---

## Design Decisions

- D1: Config source — local YAML file instead of Eidos extensionData
- D2: Loading mechanism — static loader alongside descriptors

Full decision records: `specs/issue-56-socialconfig-yaml/decisions.md`

---

## References

- SocialConfig.java — current hardcoded config with 5 characters
- ScenarioOrchestrator.java:158 — forCharacter() call site
- AgentDescriptor.java — no extensionData field (blocker for original approach)
- descriptors-baseline.yaml — descriptor YAML format for reference
- CognitiveDefaultsRegistry — does NOT handle drives/norms/beliefs (no overlap)
