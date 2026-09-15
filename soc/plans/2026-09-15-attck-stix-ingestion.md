# ATT&CK STIX Ingestion Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #56 — ATT&CK STIX ingestion — MindMap structural index + RAG prose corpus
**Issue group:** #56

**Goal:** Parse the MITRE ATT&CK Enterprise STIX 2.1 bundle at startup, ingest it into neocortex MindMap (structural graph) and RAG (prose corpus), and enrich the existing ATT&CK mapping worker with graph context (related groups, mitigations, sub-techniques).

**Architecture:** Three new classes in `io.casehub.soc.threatintel.attck` — a pure-Java STIX parser (`AttckStixParser`), a `@Startup` ingestion service (`AttckIngestionService`) that writes to MindMap + RAG, and a stateless enrichment service (`AttckEnrichmentService`) that resolves MITRE IDs via MindMap aliases. The existing `RuleAttckMappingWorker` gains enrichment by accepting `AttckEnrichmentService` as a constructor parameter.

**Tech Stack:** Java 21, Quarkus 3.32.2, Jackson (STIX JSON parsing), neocortex MindMap API (graph), neocortex RAG API (embeddings)

## Global Constraints

- Java 21 source, Java 26 JVM: `JAVA_HOME=$(/usr/libexec/java_home -v 26)`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install`
- All new classes in package `io.casehub.soc.threatintel.attck`, module `app/`
- All constants via `AttckConstants` — no magic strings
- MindMap alias system for MITRE ID resolution — no custom caches
- Version stored on dedicated root node — version stamp is last data-writing step
- `deleteCorpus` failure skips RAG ingestion (prevents duplicates) — MindMap proceeds
- Multi-module test scoping: `-pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
- Every commit references `Refs #56`

---

## Batch 1: Foundation — STIX parsing

### Task 1: Records, constants, parser, and test fixture

**Files:**
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckConstants.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckBundle.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckTechnique.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckGroup.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckMitigation.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckMalware.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckTool.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckRelationship.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckStixParser.java`
- Create: `app/src/test/resources/threatintel/test-stix-bundle.json`
- Create: `app/src/test/java/io/casehub/soc/threatintel/attck/AttckStixParserTest.java`
- Modify: `app/pom.xml` — add MindMap + RAG Maven dependencies

**Interfaces:**
- Consumes: nothing (foundation task)
- Produces:
  - `AttckConstants.REFERENCE_TENANT` → `"__reference__"` (String)
  - `AttckConstants.SUBGRAPH_NAME` → `"mitre-attack"` (String)
  - `AttckConstants.CORPUS_NAME` → `"mitre-attack"` (String)
  - `AttckStixParser.parse(InputStream) → AttckBundle`
  - `AttckBundle(String version, List<AttckTechnique>, List<AttckGroup>, List<AttckMitigation>, List<AttckMalware>, List<AttckTool>, List<AttckRelationship>)`
  - `AttckTechnique(String stixId, String mitreId, String name, String description, String detection, List<String> tactics, List<String> platforms, List<String> dataSources, String version, boolean isSubtechnique)`
  - `AttckGroup(String stixId, String mitreId, String name, String description, List<String> aliases)`
  - `AttckMitigation(String stixId, String mitreId, String name, String description)`
  - `AttckMalware(String stixId, String mitreId, String name, String description)`
  - `AttckTool(String stixId, String mitreId, String name, String description)`
  - `AttckRelationship(String stixId, String sourceRef, String targetRef, String relationshipType)`

- [ ] **Step 1: Add Maven dependencies to app/pom.xml**

Add these dependencies to `app/pom.xml` in the CaseHub Foundation section. Use `ide_replace_text_in_file` or Edit tool to insert after the existing neocortex-memory dependencies:

```xml
<!-- MindMap API + runtime impl -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-api</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-core</artifactId>
    <version>0.2-SNAPSHOT</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-sqlite</artifactId>
    <version>0.2-SNAPSHOT</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-inmem</artifactId>
    <version>0.2-SNAPSHOT</version>
    <scope>test</scope>
</dependency>

<!-- RAG API + in-memory test impl -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-rag-api</artifactId>
    <version>0.2-SNAPSHOT</version>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-rag</artifactId>
    <version>0.2-SNAPSHOT</version>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-rag-testing</artifactId>
    <version>0.2-SNAPSHOT</version>
    <scope>test</scope>
</dependency>

<!-- MindMap testing (contract test base) -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-testing</artifactId>
    <version>0.2-SNAPSHOT</version>
    <scope>test</scope>
</dependency>
```

- [ ] **Step 2: Verify dependencies resolve**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode dependency:resolve -pl app -am`
Expected: BUILD SUCCESS

- [ ] **Step 3: Create AttckConstants**

```java
package io.casehub.soc.threatintel.attck;

public final class AttckConstants {
    public static final String REFERENCE_TENANT = "__reference__";
    public static final String SUBGRAPH_NAME = "mitre-attack";
    public static final String CORPUS_NAME = "mitre-attack";
    private AttckConstants() {}
}
```

- [ ] **Step 4: Create record types**

Create each as a separate file in `io.casehub.soc.threatintel.attck`:

```java
// AttckTechnique.java
package io.casehub.soc.threatintel.attck;

import java.util.List;

public record AttckTechnique(
    String stixId, String mitreId, String name, String description,
    String detection, List<String> tactics, List<String> platforms,
    List<String> dataSources, String version, boolean isSubtechnique
) {}
```

```java
// AttckGroup.java
package io.casehub.soc.threatintel.attck;

import java.util.List;

public record AttckGroup(
    String stixId, String mitreId, String name, String description,
    List<String> aliases
) {}
```

```java
// AttckMitigation.java
package io.casehub.soc.threatintel.attck;

public record AttckMitigation(
    String stixId, String mitreId, String name, String description
) {}
```

```java
// AttckMalware.java
package io.casehub.soc.threatintel.attck;

public record AttckMalware(
    String stixId, String mitreId, String name, String description
) {}
```

```java
// AttckTool.java
package io.casehub.soc.threatintel.attck;

public record AttckTool(
    String stixId, String mitreId, String name, String description
) {}
```

```java
// AttckRelationship.java
package io.casehub.soc.threatintel.attck;

public record AttckRelationship(
    String stixId, String sourceRef, String targetRef, String relationshipType
) {}
```

```java
// AttckBundle.java
package io.casehub.soc.threatintel.attck;

import java.util.List;

public record AttckBundle(
    String version,
    List<AttckTechnique> techniques,
    List<AttckGroup> groups,
    List<AttckMitigation> mitigations,
    List<AttckMalware> malware,
    List<AttckTool> tools,
    List<AttckRelationship> relationships
) {}
```

- [ ] **Step 5: Write the parser test (failing)**

Create `app/src/test/resources/threatintel/test-stix-bundle.json` — a minimal STIX 2.1 bundle:

```json
{
  "type": "bundle",
  "id": "bundle--test",
  "objects": [
    {
      "type": "x-mitre-collection",
      "id": "x-mitre-collection--test",
      "name": "Enterprise ATT&CK",
      "x_mitre_version": "16.1"
    },
    {
      "type": "attack-pattern",
      "id": "attack-pattern--t1003",
      "name": "OS Credential Dumping",
      "description": "Adversaries may attempt to dump credentials.",
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "T1003"}
      ],
      "kill_chain_phases": [
        {"kill_chain_name": "mitre-attack", "phase_name": "credential-access"}
      ],
      "x_mitre_platforms": ["Windows", "Linux"],
      "x_mitre_data_sources": ["Process: OS API Execution"],
      "x_mitre_detection": "Monitor for unexpected processes.",
      "x_mitre_version": "1.4",
      "x_mitre_is_subtechnique": false
    },
    {
      "type": "attack-pattern",
      "id": "attack-pattern--t1003001",
      "name": "LSASS Memory",
      "description": "Adversaries may dump LSASS memory.",
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "T1003.001"}
      ],
      "kill_chain_phases": [
        {"kill_chain_name": "mitre-attack", "phase_name": "credential-access"}
      ],
      "x_mitre_platforms": ["Windows"],
      "x_mitre_data_sources": [],
      "x_mitre_version": "1.0",
      "x_mitre_is_subtechnique": true
    },
    {
      "type": "attack-pattern",
      "id": "attack-pattern--deprecated",
      "name": "Deprecated Technique",
      "description": "Should be filtered.",
      "x_mitre_deprecated": true,
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "T9999"}
      ],
      "kill_chain_phases": [],
      "x_mitre_platforms": [],
      "x_mitre_data_sources": [],
      "x_mitre_version": "1.0",
      "x_mitre_is_subtechnique": false
    },
    {
      "type": "attack-pattern",
      "id": "attack-pattern--revoked",
      "name": "Revoked Technique",
      "description": "Should be filtered.",
      "revoked": true,
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "T9998"}
      ],
      "kill_chain_phases": [],
      "x_mitre_platforms": [],
      "x_mitre_data_sources": [],
      "x_mitre_version": "1.0",
      "x_mitre_is_subtechnique": false
    },
    {
      "type": "intrusion-set",
      "id": "intrusion-set--apt28",
      "name": "APT28",
      "description": "APT28 is a threat group attributed to Russia.",
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "G0007"}
      ],
      "aliases": ["APT28", "Fancy Bear", "Sofacy"]
    },
    {
      "type": "intrusion-set",
      "id": "intrusion-set--apt29",
      "name": "APT29",
      "description": "APT29 is attributed to SVR.",
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "G0016"}
      ],
      "aliases": ["APT29", "Cozy Bear"]
    },
    {
      "type": "course-of-action",
      "id": "course-of-action--m1026",
      "name": "Privileged Account Management",
      "description": "Manage the creation and use of privileged accounts.",
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "M1026"}
      ]
    },
    {
      "type": "malware",
      "id": "malware--mimikatz",
      "name": "Mimikatz",
      "description": "Mimikatz is a credential dumper.",
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "S0002"}
      ]
    },
    {
      "type": "tool",
      "id": "tool--psexec",
      "name": "PsExec",
      "description": "PsExec is a remote execution tool.",
      "external_references": [
        {"source_name": "mitre-attack", "external_id": "S0029"}
      ]
    },
    {
      "type": "relationship",
      "id": "relationship--apt28-uses-t1003",
      "relationship_type": "uses",
      "source_ref": "intrusion-set--apt28",
      "target_ref": "attack-pattern--t1003"
    },
    {
      "type": "relationship",
      "id": "relationship--apt29-uses-t1003",
      "relationship_type": "uses",
      "source_ref": "intrusion-set--apt29",
      "target_ref": "attack-pattern--t1003"
    },
    {
      "type": "relationship",
      "id": "relationship--m1026-mitigates-t1003",
      "relationship_type": "mitigates",
      "source_ref": "course-of-action--m1026",
      "target_ref": "attack-pattern--t1003"
    },
    {
      "type": "relationship",
      "id": "relationship--subtechnique",
      "relationship_type": "subtechnique-of",
      "source_ref": "attack-pattern--t1003001",
      "target_ref": "attack-pattern--t1003"
    },
    {
      "type": "relationship",
      "id": "relationship--apt28-uses-mimikatz",
      "relationship_type": "uses",
      "source_ref": "intrusion-set--apt28",
      "target_ref": "malware--mimikatz"
    },
    {
      "type": "relationship",
      "id": "relationship--mimikatz-uses-t1003",
      "relationship_type": "uses",
      "source_ref": "malware--mimikatz",
      "target_ref": "attack-pattern--t1003"
    },
    {
      "type": "relationship",
      "id": "relationship--dangling",
      "relationship_type": "uses",
      "source_ref": "intrusion-set--unknown",
      "target_ref": "attack-pattern--t1003"
    }
  ]
}
```

Create `AttckStixParserTest.java`:

```java
package io.casehub.soc.threatintel.attck;

import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;

import java.io.InputStream;

import static org.assertj.core.api.Assertions.assertThat;

class AttckStixParserTest {

    private static AttckBundle bundle;

    @BeforeAll
    static void parseBundle() {
        InputStream is = AttckStixParserTest.class
                .getResourceAsStream("/threatintel/test-stix-bundle.json");
        assertThat(is).isNotNull();
        bundle = new AttckStixParser().parse(is);
    }

    @Test
    void extractsVersion() {
        assertThat(bundle.version()).isEqualTo("16.1");
    }

    @Test
    void parsesTechniques_excludingDeprecatedAndRevoked() {
        assertThat(bundle.techniques()).hasSize(2);
        assertThat(bundle.techniques().stream().map(AttckTechnique::mitreId))
                .containsExactlyInAnyOrder("T1003", "T1003.001");
    }

    @Test
    void techniqueFieldsExtracted() {
        var t1003 = bundle.techniques().stream()
                .filter(t -> t.mitreId().equals("T1003")).findFirst().orElseThrow();
        assertThat(t1003.stixId()).isEqualTo("attack-pattern--t1003");
        assertThat(t1003.name()).isEqualTo("OS Credential Dumping");
        assertThat(t1003.description()).startsWith("Adversaries may attempt");
        assertThat(t1003.detection()).startsWith("Monitor for unexpected");
        assertThat(t1003.tactics()).containsExactly("credential-access");
        assertThat(t1003.platforms()).containsExactlyInAnyOrder("Windows", "Linux");
        assertThat(t1003.dataSources()).containsExactly("Process: OS API Execution");
        assertThat(t1003.version()).isEqualTo("1.4");
        assertThat(t1003.isSubtechnique()).isFalse();
    }

    @Test
    void subtechniqueDetected() {
        var sub = bundle.techniques().stream()
                .filter(t -> t.mitreId().equals("T1003.001")).findFirst().orElseThrow();
        assertThat(sub.isSubtechnique()).isTrue();
    }

    @Test
    void parsesGroups() {
        assertThat(bundle.groups()).hasSize(2);
        var apt28 = bundle.groups().stream()
                .filter(g -> g.mitreId().equals("G0007")).findFirst().orElseThrow();
        assertThat(apt28.name()).isEqualTo("APT28");
        assertThat(apt28.aliases()).contains("Fancy Bear", "Sofacy");
    }

    @Test
    void parsesMitigations() {
        assertThat(bundle.mitigations()).hasSize(1);
        assertThat(bundle.mitigations().getFirst().mitreId()).isEqualTo("M1026");
    }

    @Test
    void parsesMalware() {
        assertThat(bundle.malware()).hasSize(1);
        assertThat(bundle.malware().getFirst().name()).isEqualTo("Mimikatz");
    }

    @Test
    void parsesTools() {
        assertThat(bundle.tools()).hasSize(1);
        assertThat(bundle.tools().getFirst().name()).isEqualTo("PsExec");
    }

    @Test
    void parsesRelationships() {
        assertThat(bundle.relationships()).hasSize(7);
        var usesRels = bundle.relationships().stream()
                .filter(r -> r.relationshipType().equals("uses")).toList();
        assertThat(usesRels).hasSize(4);
    }

    @Test
    void relationshipFieldsExtracted() {
        var rel = bundle.relationships().stream()
                .filter(r -> r.stixId().equals("relationship--apt28-uses-t1003"))
                .findFirst().orElseThrow();
        assertThat(rel.sourceRef()).isEqualTo("intrusion-set--apt28");
        assertThat(rel.targetRef()).isEqualTo("attack-pattern--t1003");
        assertThat(rel.relationshipType()).isEqualTo("uses");
    }
}
```

- [ ] **Step 6: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=AttckStixParserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `AttckStixParser` not found

- [ ] **Step 7: Implement AttckStixParser**

```java
package io.casehub.soc.threatintel.attck;

import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.util.ArrayList;
import java.util.List;

public class AttckStixParser {

    private static final ObjectMapper MAPPER = new ObjectMapper();

    public AttckBundle parse(InputStream stixJson) {
        try {
            JsonNode root = MAPPER.readTree(stixJson);
            JsonNode objects = root.get("objects");
            if (objects == null || !objects.isArray()) {
                return new AttckBundle("", List.of(), List.of(), List.of(), List.of(), List.of(), List.of());
            }

            String version = "";
            List<AttckTechnique> techniques = new ArrayList<>();
            List<AttckGroup> groups = new ArrayList<>();
            List<AttckMitigation> mitigations = new ArrayList<>();
            List<AttckMalware> malware = new ArrayList<>();
            List<AttckTool> tools = new ArrayList<>();
            List<AttckRelationship> relationships = new ArrayList<>();

            for (JsonNode obj : objects) {
                if (isDeprecatedOrRevoked(obj)) continue;

                String type = textOrEmpty(obj, "type");
                switch (type) {
                    case "x-mitre-collection" -> version = textOrEmpty(obj, "x_mitre_version");
                    case "attack-pattern" -> techniques.add(parseTechnique(obj));
                    case "intrusion-set" -> groups.add(parseGroup(obj));
                    case "course-of-action" -> mitigations.add(parseMitigation(obj));
                    case "malware" -> malware.add(parseMalware(obj));
                    case "tool" -> tools.add(parseTool(obj));
                    case "relationship" -> relationships.add(parseRelationship(obj));
                }
            }
            return new AttckBundle(version, techniques, groups, mitigations, malware, tools, relationships);
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to parse STIX bundle", e);
        }
    }

    private boolean isDeprecatedOrRevoked(JsonNode obj) {
        return boolOrFalse(obj, "x_mitre_deprecated") || boolOrFalse(obj, "revoked");
    }

    private AttckTechnique parseTechnique(JsonNode obj) {
        return new AttckTechnique(
                textOrEmpty(obj, "id"),
                extractMitreId(obj),
                textOrEmpty(obj, "name"),
                textOrEmpty(obj, "description"),
                obj.has("x_mitre_detection") ? obj.get("x_mitre_detection").asText() : null,
                extractKillChainPhases(obj),
                textList(obj, "x_mitre_platforms"),
                textList(obj, "x_mitre_data_sources"),
                textOrEmpty(obj, "x_mitre_version"),
                boolOrFalse(obj, "x_mitre_is_subtechnique")
        );
    }

    private AttckGroup parseGroup(JsonNode obj) {
        return new AttckGroup(
                textOrEmpty(obj, "id"),
                extractMitreId(obj),
                textOrEmpty(obj, "name"),
                textOrEmpty(obj, "description"),
                textList(obj, "aliases")
        );
    }

    private AttckMitigation parseMitigation(JsonNode obj) {
        return new AttckMitigation(
                textOrEmpty(obj, "id"),
                extractMitreId(obj),
                textOrEmpty(obj, "name"),
                textOrEmpty(obj, "description")
        );
    }

    private AttckMalware parseMalware(JsonNode obj) {
        return new AttckMalware(
                textOrEmpty(obj, "id"),
                extractMitreId(obj),
                textOrEmpty(obj, "name"),
                textOrEmpty(obj, "description")
        );
    }

    private AttckTool parseTool(JsonNode obj) {
        return new AttckTool(
                textOrEmpty(obj, "id"),
                extractMitreId(obj),
                textOrEmpty(obj, "name"),
                textOrEmpty(obj, "description")
        );
    }

    private AttckRelationship parseRelationship(JsonNode obj) {
        return new AttckRelationship(
                textOrEmpty(obj, "id"),
                textOrEmpty(obj, "source_ref"),
                textOrEmpty(obj, "target_ref"),
                textOrEmpty(obj, "relationship_type")
        );
    }

    private String extractMitreId(JsonNode obj) {
        JsonNode refs = obj.get("external_references");
        if (refs != null && refs.isArray()) {
            for (JsonNode ref : refs) {
                if ("mitre-attack".equals(textOrEmpty(ref, "source_name"))) {
                    return textOrEmpty(ref, "external_id");
                }
            }
        }
        return "";
    }

    private List<String> extractKillChainPhases(JsonNode obj) {
        JsonNode phases = obj.get("kill_chain_phases");
        if (phases == null || !phases.isArray()) return List.of();
        List<String> result = new ArrayList<>();
        for (JsonNode phase : phases) {
            if ("mitre-attack".equals(textOrEmpty(phase, "kill_chain_name"))) {
                result.add(textOrEmpty(phase, "phase_name"));
            }
        }
        return result;
    }

    private List<String> textList(JsonNode obj, String field) {
        JsonNode arr = obj.get(field);
        if (arr == null || !arr.isArray()) return List.of();
        List<String> result = new ArrayList<>();
        for (JsonNode item : arr) {
            result.add(item.asText());
        }
        return result;
    }

    private String textOrEmpty(JsonNode obj, String field) {
        JsonNode node = obj.get(field);
        return node != null ? node.asText() : "";
    }

    private boolean boolOrFalse(JsonNode obj, String field) {
        JsonNode node = obj.get(field);
        return node != null && node.asBoolean(false);
    }
}
```

- [ ] **Step 8: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=AttckStixParserTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — all 9 tests green

- [ ] **Step 9: Commit**

```bash
git add app/pom.xml app/src/main/java/io/casehub/soc/threatintel/ app/src/test/java/io/casehub/soc/threatintel/ app/src/test/resources/threatintel/
git commit -m "feat(#56): add ATT&CK STIX 2.1 parser with records and constants

Pure-Java parser for MITRE ATT&CK Enterprise STIX bundle. Extracts
techniques, groups, mitigations, malware, tools, and relationships.
Filters deprecated/revoked objects. Adds neocortex MindMap + RAG
Maven dependencies.

Refs #56"
```

---

## Batch 2: Ingestion + Enrichment — data pipeline

### Task 2: AttckEnrichmentService + unit test

**Files:**
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckRelatedEntity.java`
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckEnrichmentService.java`
- Create: `app/src/test/java/io/casehub/soc/threatintel/attck/AttckEnrichmentServiceTest.java`

**Interfaces:**
- Consumes: `MindMapStore` (from neocortex-mindmap-api), `AttckConstants`
- Produces:
  - `AttckEnrichmentService.getRelatedGroups(String mitreId) → List<AttckRelatedEntity>`
  - `AttckEnrichmentService.getMitigations(String mitreId) → List<AttckRelatedEntity>`
  - `AttckEnrichmentService.getSubTechniques(String mitreId) → List<AttckRelatedEntity>`
  - `AttckEnrichmentService.getGroupTechniques(String groupMitreId) → List<AttckRelatedEntity>`
  - `AttckRelatedEntity(String mitreId, String name, String trait)`
  - `AttckRelatedEntity.from(MindMapNode) → AttckRelatedEntity`

- [ ] **Step 1: Write the enrichment service test (failing)**

```java
package io.casehub.soc.threatintel.attck;

import io.casehub.neocortex.mindmap.MindMapEdge;
import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;

import java.util.List;
import java.util.Map;
import java.util.Optional;
import java.util.Set;

import static io.casehub.soc.threatintel.attck.AttckConstants.REFERENCE_TENANT;
import static org.assertj.core.api.Assertions.assertThat;
import static org.mockito.Mockito.*;

class AttckEnrichmentServiceTest {

    private MindMapStore store;
    private AttckEnrichmentService service;

    @BeforeEach
    void setUp() {
        store = mock(MindMapStore.class);
        service = new AttckEnrichmentService();
        service.mindMapStore = store;
    }

    @Test
    void getRelatedGroups_returnsInboundUsesEdgesWithThreatGroupTrait() {
        var techNode = mockNode("node-t1003", "T1003", "OS Credential Dumping", "attack-technique");
        var groupNode = mockNode("node-g0007", "G0007", "APT28", "threat-group");
        var edge = mockEdge("node-g0007", "node-t1003", "uses");

        when(store.resolveNode("T1003", null, REFERENCE_TENANT)).thenReturn(techNode);
        when(store.neighbors("node-t1003", "uses", REFERENCE_TENANT)).thenReturn(List.of(edge));
        when(store.getNode("node-g0007", REFERENCE_TENANT)).thenReturn(groupNode);

        var result = service.getRelatedGroups("T1003");

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().mitreId()).isEqualTo("G0007");
        assertThat(result.getFirst().name()).isEqualTo("APT28");
        assertThat(result.getFirst().trait()).isEqualTo("threat-group");
    }

    @Test
    void getRelatedGroups_filtersOutNonGroupTraits() {
        var techNode = mockNode("node-t1003", "T1003", "OS Credential Dumping", "attack-technique");
        var malwareNode = mockNode("node-s0002", "S0002", "Mimikatz", "malware");
        var edge = mockEdge("node-s0002", "node-t1003", "uses");

        when(store.resolveNode("T1003", null, REFERENCE_TENANT)).thenReturn(techNode);
        when(store.neighbors("node-t1003", "uses", REFERENCE_TENANT)).thenReturn(List.of(edge));
        when(store.getNode("node-s0002", REFERENCE_TENANT)).thenReturn(malwareNode);

        var result = service.getRelatedGroups("T1003");
        assertThat(result).isEmpty();
    }

    @Test
    void getMitigations_returnsInboundMitigatesEdges() {
        var techNode = mockNode("node-t1003", "T1003", "OS Credential Dumping", "attack-technique");
        var mitNode = mockNode("node-m1026", "M1026", "Privileged Account Management", "mitigation");
        var edge = mockEdge("node-m1026", "node-t1003", "mitigates");

        when(store.resolveNode("T1003", null, REFERENCE_TENANT)).thenReturn(techNode);
        when(store.neighbors("node-t1003", "mitigates", REFERENCE_TENANT)).thenReturn(List.of(edge));
        when(store.getNode("node-m1026", REFERENCE_TENANT)).thenReturn(mitNode);

        var result = service.getMitigations("T1003");

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().mitreId()).isEqualTo("M1026");
    }

    @Test
    void getSubTechniques_returnsInboundSubtechniqueOfEdges() {
        var parentNode = mockNode("node-t1003", "T1003", "OS Credential Dumping", "attack-technique");
        var subNode = mockNode("node-t1003001", "T1003.001", "LSASS Memory", "attack-technique");
        var edge = mockEdge("node-t1003001", "node-t1003", "subtechnique-of");

        when(store.resolveNode("T1003", null, REFERENCE_TENANT)).thenReturn(parentNode);
        when(store.neighbors("node-t1003", "subtechnique-of", REFERENCE_TENANT)).thenReturn(List.of(edge));
        when(store.getNode("node-t1003001", REFERENCE_TENANT)).thenReturn(subNode);

        var result = service.getSubTechniques("T1003");

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().mitreId()).isEqualTo("T1003.001");
    }

    @Test
    void getGroupTechniques_returnsOutboundUsesEdges() {
        var groupNode = mockNode("node-g0007", "G0007", "APT28", "threat-group");
        var techNode = mockNode("node-t1003", "T1003", "OS Credential Dumping", "attack-technique");
        var edge = mockEdge("node-g0007", "node-t1003", "uses");

        when(store.resolveNode("G0007", null, REFERENCE_TENANT)).thenReturn(groupNode);
        when(store.neighbors("node-g0007", "uses", REFERENCE_TENANT)).thenReturn(List.of(edge));
        when(store.getNode("node-t1003", REFERENCE_TENANT)).thenReturn(techNode);

        var result = service.getGroupTechniques("G0007");

        assertThat(result).hasSize(1);
        assertThat(result.getFirst().mitreId()).isEqualTo("T1003");
    }

    @Test
    void unknownMitreId_returnsEmptyList() {
        when(store.resolveNode("TXXX", null, REFERENCE_TENANT)).thenReturn(null);

        assertThat(service.getRelatedGroups("TXXX")).isEmpty();
        assertThat(service.getMitigations("TXXX")).isEmpty();
        assertThat(service.getSubTechniques("TXXX")).isEmpty();
        assertThat(service.getGroupTechniques("TXXX")).isEmpty();
    }

    @Test
    void nullNode_fromGetNode_filteredOut() {
        var techNode = mockNode("node-t1003", "T1003", "OS Credential Dumping", "attack-technique");
        var edge = mockEdge("node-deleted", "node-t1003", "uses");

        when(store.resolveNode("T1003", null, REFERENCE_TENANT)).thenReturn(techNode);
        when(store.neighbors("node-t1003", "uses", REFERENCE_TENANT)).thenReturn(List.of(edge));
        when(store.getNode("node-deleted", REFERENCE_TENANT)).thenReturn(null);

        assertThat(service.getRelatedGroups("T1003")).isEmpty();
    }

    private MindMapNode mockNode(String id, String mitreId, String name, String trait) {
        MindMapNode node = mock(MindMapNode.class);
        when(node.id()).thenReturn(id);
        when(node.name()).thenReturn(name);
        when(node.traits()).thenReturn(Set.of(trait));
        when(node.property("mitreId")).thenReturn(Optional.of(mitreId));
        return node;
    }

    private MindMapEdge mockEdge(String sourceId, String targetId, String edgeType) {
        MindMapEdge edge = mock(MindMapEdge.class);
        when(edge.sourceNodeId()).thenReturn(sourceId);
        when(edge.targetNodeId()).thenReturn(targetId);
        when(edge.edgeType()).thenReturn(edgeType);
        return edge;
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=AttckEnrichmentServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `AttckEnrichmentService` not found

- [ ] **Step 3: Implement AttckRelatedEntity**

```java
package io.casehub.soc.threatintel.attck;

import io.casehub.neocortex.mindmap.MindMapNode;

import java.util.Set;

public record AttckRelatedEntity(String mitreId, String name, String trait) {

    private static final Set<String> ATTCK_TRAITS = Set.of(
            "attack-technique", "threat-group", "mitigation", "malware", "attack-tool", "metadata");

    static AttckRelatedEntity from(MindMapNode node) {
        String trait = node.traits().stream()
                .filter(ATTCK_TRAITS::contains)
                .findFirst().orElse("");
        return new AttckRelatedEntity(
                node.property("mitreId").orElse(""),
                node.name(),
                trait);
    }
}
```

- [ ] **Step 4: Implement AttckEnrichmentService**

```java
package io.casehub.soc.threatintel.attck;

import io.casehub.neocortex.mindmap.MindMapNode;
import io.casehub.neocortex.mindmap.MindMapStore;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.inject.Inject;

import java.util.List;
import java.util.Objects;

import static io.casehub.soc.threatintel.attck.AttckConstants.REFERENCE_TENANT;

@ApplicationScoped
public class AttckEnrichmentService {

    @Inject
    MindMapStore mindMapStore;

    public List<AttckRelatedEntity> getRelatedGroups(String mitreId) {
        MindMapNode node = mindMapStore.resolveNode(mitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "uses", REFERENCE_TENANT).stream()
                .filter(edge -> edge.targetNodeId().equals(node.id()))
                .map(edge -> mindMapStore.getNode(edge.sourceNodeId(), REFERENCE_TENANT))
                .filter(n -> n != null && n.traits().contains("threat-group"))
                .map(AttckRelatedEntity::from)
                .toList();
    }

    public List<AttckRelatedEntity> getMitigations(String mitreId) {
        MindMapNode node = mindMapStore.resolveNode(mitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "mitigates", REFERENCE_TENANT).stream()
                .filter(edge -> edge.targetNodeId().equals(node.id()))
                .map(edge -> mindMapStore.getNode(edge.sourceNodeId(), REFERENCE_TENANT))
                .filter(n -> n != null && n.traits().contains("mitigation"))
                .map(AttckRelatedEntity::from)
                .toList();
    }

    public List<AttckRelatedEntity> getSubTechniques(String mitreId) {
        MindMapNode node = mindMapStore.resolveNode(mitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "subtechnique-of", REFERENCE_TENANT).stream()
                .filter(edge -> edge.targetNodeId().equals(node.id()))
                .map(edge -> mindMapStore.getNode(edge.sourceNodeId(), REFERENCE_TENANT))
                .filter(Objects::nonNull)
                .map(AttckRelatedEntity::from)
                .toList();
    }

    public List<AttckRelatedEntity> getGroupTechniques(String groupMitreId) {
        MindMapNode node = mindMapStore.resolveNode(groupMitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "uses", REFERENCE_TENANT).stream()
                .filter(edge -> edge.sourceNodeId().equals(node.id()))
                .map(edge -> mindMapStore.getNode(edge.targetNodeId(), REFERENCE_TENANT))
                .filter(n -> n != null && n.traits().contains("attack-technique"))
                .map(AttckRelatedEntity::from)
                .toList();
    }
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=AttckEnrichmentServiceTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — all 7 tests green

- [ ] **Step 6: Commit**

```bash
git add app/src/main/java/io/casehub/soc/threatintel/attck/AttckRelatedEntity.java app/src/main/java/io/casehub/soc/threatintel/attck/AttckEnrichmentService.java app/src/test/java/io/casehub/soc/threatintel/attck/AttckEnrichmentServiceTest.java
git commit -m "feat(#56): add ATT&CK enrichment service with graph queries

Stateless service resolving MITRE IDs via MindMap alias system.
Queries for related groups, mitigations, sub-techniques, and
group techniques via neighbors() with direction/trait filtering.

Refs #56"
```

### Task 3: AttckIngestionService + integration test

**Files:**
- Create: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckIngestionService.java`
- Modify: `app/src/main/resources/application.properties` — add `casehub.soc.attck.enabled=true`
- Create: `app/src/test/java/io/casehub/soc/threatintel/attck/AttckIngestionIntegrationTest.java`

**Interfaces:**
- Consumes: `AttckStixParser` (Task 1), `AttckConstants` (Task 1), `MindMapStore`, `EmbeddingIngestor`
- Produces: MindMap subgraph `mitre-attack` with nodes/edges/aliases + RAG corpus `mitre-attack` with prose chunks

- [ ] **Step 1: Write the integration test (failing)**

Create `AttckIngestionIntegrationTest.java`. This test class exercises the full pipeline using in-memory MindMap and RAG backends. Each test method covers one spec scenario:

```java
package io.casehub.soc.threatintel.attck;

import io.casehub.neocortex.mindmap.MindMapStore;
import io.casehub.neocortex.rag.CorpusRef;
import io.casehub.neocortex.rag.EmbeddingIngestor;
import io.casehub.neocortex.rag.CaseRetriever;
import io.quarkus.test.junit.QuarkusTest;
import jakarta.inject.Inject;
import org.junit.jupiter.api.Test;

import static io.casehub.soc.threatintel.attck.AttckConstants.*;
import static org.assertj.core.api.Assertions.assertThat;

@QuarkusTest
class AttckIngestionIntegrationTest {

    @Inject MindMapStore mindMapStore;
    @Inject AttckEnrichmentService enrichmentService;

    @Test
    void nodesExistWithCorrectTraitsAndProperties() {
        var subgraphs = mindMapStore.listSubgraphs(REFERENCE_TENANT);
        var sg = subgraphs.stream()
                .filter(s -> s.name().equals(SUBGRAPH_NAME))
                .findFirst().orElseThrow();
        var nodes = mindMapStore.nodesIn(sg.id(), REFERENCE_TENANT);
        assertThat(nodes).isNotEmpty();

        var t1003 = mindMapStore.resolveNode("T1003", null, REFERENCE_TENANT);
        assertThat(t1003).isNotNull();
        assertThat(t1003.traits()).contains("attack-technique");
        assertThat(t1003.property("mitreId")).hasValue("T1003");
    }

    @Test
    void edgesExistForRelationships() {
        var groups = enrichmentService.getRelatedGroups("T1003");
        assertThat(groups).isNotEmpty();
        assertThat(groups.stream().map(AttckRelatedEntity::name))
                .contains("APT28");
    }

    @Test
    void enrichmentServiceReturnsCorrectResults() {
        assertThat(enrichmentService.getMitigations("T1003")).isNotEmpty();
        assertThat(enrichmentService.getSubTechniques("T1003")).isNotEmpty();
        assertThat(enrichmentService.getGroupTechniques("G0007")).isNotEmpty();
    }

    @Test
    void aliasRegistration() {
        assertThat(mindMapStore.resolveNode("T1003", null, REFERENCE_TENANT)).isNotNull();
        assertThat(mindMapStore.resolveNode("G0007", null, REFERENCE_TENANT)).isNotNull();
        assertThat(mindMapStore.resolveNode("M1026", null, REFERENCE_TENANT)).isNotNull();
        assertThat(mindMapStore.resolveNode("S0002", null, REFERENCE_TENANT)).isNotNull();
        assertThat(mindMapStore.resolveNode("S0029", null, REFERENCE_TENANT)).isNotNull();
    }

    @Test
    void rootNodeVersionStorage() {
        var subgraphs = mindMapStore.listSubgraphs(REFERENCE_TENANT);
        var sg = subgraphs.stream()
                .filter(s -> s.name().equals(SUBGRAPH_NAME))
                .findFirst().orElseThrow();
        assertThat(sg.rootNodeId()).isNotNull();
        var rootNode = mindMapStore.getNode(sg.rootNodeId(), REFERENCE_TENANT);
        assertThat(rootNode).isNotNull();
        assertThat(rootNode.property("attck-version")).hasValue("16.1");
    }
}
```

Note: Additional resilience tests (crash recovery, error boundary, RAG failure isolation, NoOp guard, deleteCorpus failure, edge drop logging) should be added as separate test methods in this class. They require mock injection or test-profile configuration — implement them after the core pipeline is green.

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=AttckIngestionIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `AttckIngestionService` not found or ingestion does not run

- [ ] **Step 3: Add application.properties config**

Add to `app/src/main/resources/application.properties`:
```properties
casehub.soc.attck.enabled=true
```

- [ ] **Step 4: Copy the test STIX bundle to test resources for integration test**

The integration test uses the same test fixture as the parser test (`app/src/test/resources/threatintel/test-stix-bundle.json`). The `AttckIngestionService` loads from `/threatintel/enterprise-attack.json` on the classpath. Create a test-profile override or symlink so the integration test finds the bundle. The simplest approach: copy the test bundle to `app/src/test/resources/threatintel/enterprise-attack.json` so it's on the test classpath at the path the service expects.

```bash
cp app/src/test/resources/threatintel/test-stix-bundle.json app/src/test/resources/threatintel/enterprise-attack.json
```

- [ ] **Step 5: Implement AttckIngestionService**

Create the full service as specified in the design spec (§AttckIngestionService). The service:
1. Checks `enabled` config flag
2. Checks `MindMapCapability.SUBGRAPH` capability
3. Loads and parses STIX bundle from classpath
4. Version-checks via subgraph root node
5. Erases old subgraph if re-ingesting
6. Handles corpus cleanup with failure flag
7. Registers vocabulary (idempotent)
8. Creates subgraph, nodes with aliases, and edges
9. Ingests RAG corpus (techniques, groups, mitigations, malware, tools)
10. Creates root node with version stamp (last step)
11. Wraps everything in try-catch for error boundary

```java
package io.casehub.soc.threatintel.attck;

import io.casehub.neocortex.mindmap.*;
import io.casehub.neocortex.rag.ChunkInput;
import io.casehub.neocortex.rag.CorpusRef;
import io.casehub.neocortex.rag.EmbeddingIngestor;
import io.quarkus.logging.Log;
import io.quarkus.runtime.StartupEvent;
import jakarta.enterprise.context.ApplicationScoped;
import jakarta.enterprise.event.Observes;
import jakarta.inject.Inject;
import org.eclipse.microprofile.config.inject.ConfigProperty;

import java.util.*;
import java.util.stream.Collectors;

import static io.casehub.soc.threatintel.attck.AttckConstants.*;

@ApplicationScoped
public class AttckIngestionService {

    @ConfigProperty(name = "casehub.soc.attck.enabled", defaultValue = "true")
    boolean enabled;

    @Inject MindMapStore mindMapStore;
    @Inject EmbeddingIngestor embeddingIngestor;

    void onStartup(@Observes StartupEvent event) {
        if (!enabled) {
            Log.info("ATT&CK ingestion disabled via config");
            return;
        }

        if (!mindMapStore.capabilities().contains(MindMapCapability.SUBGRAPH)) {
            Log.info("MindMapStore does not support SUBGRAPH — skipping ATT&CK ingestion");
            return;
        }

        try {
            var stixStream = getClass().getResourceAsStream("/threatintel/enterprise-attack.json");
            if (stixStream == null) {
                Log.warn("ATT&CK STIX bundle not found on classpath — skipping ingestion");
                return;
            }

            var subgraphs = mindMapStore.listSubgraphs(REFERENCE_TENANT);
            var existing = subgraphs.stream()
                    .filter(s -> s.name().equals(SUBGRAPH_NAME))
                    .findFirst().orElse(null);
            String storedVersion = null;
            if (existing != null && existing.rootNodeId() != null) {
                var rootNode = mindMapStore.getNode(existing.rootNodeId(), REFERENCE_TENANT);
                if (rootNode != null) {
                    storedVersion = rootNode.property("attck-version").orElse(null);
                }
            }

            AttckBundle bundle = new AttckStixParser().parse(stixStream);
            if (bundle.version().equals(storedVersion)) return;

            if (existing != null) {
                mindMapStore.eraseSubgraph(existing.id(), REFERENCE_TENANT);
            }

            CorpusRef corpus = new CorpusRef(REFERENCE_TENANT, CORPUS_NAME);
            boolean corpusCleanupFailed = false;
            try {
                embeddingIngestor.deleteCorpus(corpus);
            } catch (Exception e) {
                Log.warn("deleteCorpus failed — RAG re-ingestion will be skipped to prevent duplicates", e);
                corpusCleanupFailed = true;
            }

            mindMapStore.registerVocabulary(MindMapVocabulary.builder()
                    .edgeType("uses").edgeType("mitigates").edgeType("subtechnique-of")
                    .build());

            String subgraphId = mindMapStore.createSubgraph(
                    new SubgraphInput(SUBGRAPH_NAME, SUBGRAPH_NAME, null),
                    REFERENCE_TENANT);

            Map<String, String> stixToStoreId = new HashMap<>();
            for (var t : bundle.techniques()) {
                String storeId = mindMapStore.addNode(
                        NodeInput.of(t.name(), subgraphId)
                                .withTraits(Set.of("attack-technique"))
                                .withRefs(Set.of(new NodeRef("mitre-attack", t.mitreId(), null)))
                                .withProperties(Map.of(
                                        "mitreId", t.mitreId(),
                                        "name", t.name(),
                                        "tactics", String.join(",", t.tactics()),
                                        "platforms", String.join(",", t.platforms()),
                                        "dataSources", String.join(",", t.dataSources()),
                                        "version", t.version(),
                                        "isSubtechnique", String.valueOf(t.isSubtechnique()))),
                        REFERENCE_TENANT);
                mindMapStore.addAlias(storeId, t.mitreId(), REFERENCE_TENANT);
                stixToStoreId.put(t.stixId(), storeId);
            }
            for (var g : bundle.groups()) {
                String storeId = mindMapStore.addNode(
                        NodeInput.of(g.name(), subgraphId)
                                .withTraits(Set.of("threat-group"))
                                .withRefs(Set.of(new NodeRef("mitre-attack", g.mitreId(), null)))
                                .withProperties(Map.of(
                                        "mitreId", g.mitreId(),
                                        "name", g.name(),
                                        "aliases", String.join(",", g.aliases()))),
                        REFERENCE_TENANT);
                mindMapStore.addAlias(storeId, g.mitreId(), REFERENCE_TENANT);
                stixToStoreId.put(g.stixId(), storeId);
            }
            for (var m : bundle.mitigations()) {
                String storeId = mindMapStore.addNode(
                        NodeInput.of(m.name(), subgraphId)
                                .withTraits(Set.of("mitigation"))
                                .withRefs(Set.of(new NodeRef("mitre-attack", m.mitreId(), null)))
                                .withProperties(Map.of("mitreId", m.mitreId(), "name", m.name())),
                        REFERENCE_TENANT);
                mindMapStore.addAlias(storeId, m.mitreId(), REFERENCE_TENANT);
                stixToStoreId.put(m.stixId(), storeId);
            }
            for (var mw : bundle.malware()) {
                String storeId = mindMapStore.addNode(
                        NodeInput.of(mw.name(), subgraphId)
                                .withTraits(Set.of("malware"))
                                .withRefs(Set.of(new NodeRef("mitre-attack", mw.mitreId(), null)))
                                .withProperties(Map.of("mitreId", mw.mitreId(), "name", mw.name())),
                        REFERENCE_TENANT);
                mindMapStore.addAlias(storeId, mw.mitreId(), REFERENCE_TENANT);
                stixToStoreId.put(mw.stixId(), storeId);
            }
            for (var tool : bundle.tools()) {
                String storeId = mindMapStore.addNode(
                        NodeInput.of(tool.name(), subgraphId)
                                .withTraits(Set.of("attack-tool"))
                                .withRefs(Set.of(new NodeRef("mitre-attack", tool.mitreId(), null)))
                                .withProperties(Map.of("mitreId", tool.mitreId(), "name", tool.name())),
                        REFERENCE_TENANT);
                mindMapStore.addAlias(storeId, tool.mitreId(), REFERENCE_TENANT);
                stixToStoreId.put(tool.stixId(), storeId);
            }

            int droppedSrcMissing = 0, droppedTgtMissing = 0;
            for (var rel : bundle.relationships()) {
                String src = stixToStoreId.get(rel.sourceRef());
                String tgt = stixToStoreId.get(rel.targetRef());
                if (src != null && tgt != null) {
                    mindMapStore.addEdge(EdgeInput.of(src, tgt, rel.relationshipType()), REFERENCE_TENANT);
                } else {
                    if (src == null) droppedSrcMissing++;
                    if (tgt == null) droppedTgtMissing++;
                }
            }
            if (droppedSrcMissing + droppedTgtMissing > 0) {
                Log.infof("ATT&CK edges dropped: %d (source missing: %d, target missing: %d)",
                        droppedSrcMissing + droppedTgtMissing, droppedSrcMissing, droppedTgtMissing);
            }

            if (!corpusCleanupFailed) {
                try {
                    embeddingIngestor.ingest(corpus, buildTechniqueChunks(bundle.techniques()));
                    embeddingIngestor.ingest(corpus, buildGroupChunks(bundle.groups()));
                    embeddingIngestor.ingest(corpus, buildMitigationChunks(bundle.mitigations()));
                    embeddingIngestor.ingest(corpus, buildMalwareChunks(bundle.malware()));
                    embeddingIngestor.ingest(corpus, buildToolChunks(bundle.tools()));
                } catch (Exception e) {
                    Log.warn("RAG ingestion failed — MindMap available, prose retrieval unavailable", e);
                }
            } else {
                Log.warn("RAG re-ingestion skipped — stale RAG data may remain until next successful re-ingestion");
            }

            String rootId = mindMapStore.addNode(
                    NodeInput.of("mitre-attack-root", subgraphId)
                            .withTraits(Set.of("metadata"))
                            .withProperties(Map.of("attck-version", bundle.version())),
                    REFERENCE_TENANT);
            mindMapStore.updateSubgraph(subgraphId, rootId, REFERENCE_TENANT);

        } catch (Exception e) {
            Log.error("ATT&CK ingestion failed — enrichment unavailable, static lookup active", e);
        }
    }

    private List<ChunkInput> buildTechniqueChunks(List<AttckTechnique> techniques) {
        return techniques.stream().map(t -> {
            String content = t.detection() != null
                    ? t.description() + "\n\n" + t.detection()
                    : t.description();
            return new ChunkInput(content, t.mitreId(), Map.of(
                    "mitreId", t.mitreId(), "name", t.name(),
                    "tactics", String.join(",", t.tactics()), "type", "attack-technique"));
        }).toList();
    }

    private List<ChunkInput> buildGroupChunks(List<AttckGroup> groups) {
        return groups.stream().map(g -> new ChunkInput(
                g.description(), g.mitreId(), Map.of(
                        "mitreId", g.mitreId(), "name", g.name(), "type", "threat-group")))
                .toList();
    }

    private List<ChunkInput> buildMitigationChunks(List<AttckMitigation> mitigations) {
        return mitigations.stream().map(m -> new ChunkInput(
                m.description(), m.mitreId(), Map.of(
                        "mitreId", m.mitreId(), "name", m.name(), "type", "mitigation")))
                .toList();
    }

    private List<ChunkInput> buildMalwareChunks(List<AttckMalware> malware) {
        return malware.stream().map(m -> new ChunkInput(
                m.description(), m.mitreId(), Map.of(
                        "mitreId", m.mitreId(), "name", m.name(), "type", "malware")))
                .toList();
    }

    private List<ChunkInput> buildToolChunks(List<AttckTool> tools) {
        return tools.stream().map(t -> new ChunkInput(
                t.description(), t.mitreId(), Map.of(
                        "mitreId", t.mitreId(), "name", t.name(), "type", "attack-tool")))
                .toList();
    }
}
```

- [ ] **Step 6: Run integration tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=AttckIngestionIntegrationTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — all 5 tests green

- [ ] **Step 7: Commit**

```bash
git add app/src/main/java/io/casehub/soc/threatintel/attck/AttckIngestionService.java app/src/main/resources/application.properties app/src/test/java/io/casehub/soc/threatintel/attck/AttckIngestionIntegrationTest.java app/src/test/resources/threatintel/enterprise-attack.json
git commit -m "feat(#56): add ATT&CK startup ingestion service with integration tests

@Startup service parses STIX 2.1, populates MindMap subgraph with
nodes/aliases/edges, ingests RAG corpus for techniques/groups/
mitigations/malware/tools. Version-stamped root node enables
idempotent crash recovery. Error boundary prevents startup failure.

Refs #56"
```

---

## Batch 3: Worker wiring — enriched ATT&CK output

### Task 4: Package consolidation + worker enhancement + descriptor wiring

**Files:**
- Move: `../../app/src/main/java/io/casehub/soc/threatintel/attck/AttckLookupTable.java` → `app/src/main/java/io/casehub/soc/threatintel/attck/AttckLookupTable.java` (use `ide_move_file`)
- Modify: `app/src/main/java/io/casehub/soc/worker/RuleAttckMappingWorker.java` — add `AttckEnrichmentService` parameter, `enrichTechnique()` helper
- Modify: `app/src/main/java/io/casehub/soc/engine/SocInvestigationCaseDescriptor.java` — add `AttckEnrichmentService` constructor parameter
- Modify: `app/src/main/java/io/casehub/soc/engine/SocCaseHub.java` — inject `AttckEnrichmentService`
- Modify: `app/src/test/java/io/casehub/soc/worker/RuleAttckMappingWorkerTest.java` — update for new `create()` signature
- Modify: `app/src/test/java/io/casehub/soc/engine/SocInvestigationCaseDescriptorTest.java` — update constructor call
- Modify: `app/src/test/java/io/casehub/soc/engine/SocCaseHubTest.java` — update if needed

**Interfaces:**
- Consumes: `AttckEnrichmentService` (Task 2), `AttckLookupTable`, `AttckMappingOutput.TechniqueEntry`
- Produces: Enriched worker output with `relatedGroups`, `mitigations`, `subTechniques` fields

- [ ] **Step 1: Move AttckLookupTable to new package**

Use `ide_move_file`:
- From: `../../app/src/main/java/io/casehub/soc/threatintel/attck/AttckLookupTable.java`
- To: `app/src/main/java/io/casehub/soc/threatintel/attck/AttckLookupTable.java`

This updates the package declaration and all import references automatically.

- [ ] **Step 2: Verify move compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode compile -pl app -am`
Expected: BUILD SUCCESS

- [ ] **Step 3: Update RuleAttckMappingWorker test for new signature (failing)**

Update `RuleAttckMappingWorkerTest.java`:

```java
// Add import
import io.casehub.soc.threatintel.attck.AttckEnrichmentService;
import static org.mockito.Mockito.mock;

// Change worker creation
private final AttckEnrichmentService enrichmentService = mock(AttckEnrichmentService.class);
private final Worker worker = RuleAttckMappingWorker.create(enrichmentService);
```

- [ ] **Step 4: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dtest=RuleAttckMappingWorkerTest -Dsurefire.failIfNoSpecifiedTests=false`
Expected: FAIL — `create()` does not accept `AttckEnrichmentService`

- [ ] **Step 5: Update RuleAttckMappingWorker to accept AttckEnrichmentService**

Replace `RuleAttckMappingWorker.create()`:

```java
package io.casehub.soc.worker;

import io.casehub.soc.threatintel.attck.AttckEnrichmentService;
import io.casehub.soc.threatintel.attck.AttckLookupTable;
import io.casehub.soc.threatintel.attck.AttckRelatedEntity;
import io.casehub.soc.worker.contract.AttckMappingOutput;
import io.casehub.worker.api.Worker;
import io.casehub.worker.api.WorkerResult;

import java.util.List;
import java.util.Map;
import java.util.stream.Collectors;

public final class RuleAttckMappingWorker {

    private RuleAttckMappingWorker() {}

    public static Worker create(AttckEnrichmentService enrichmentService) {
        return Worker.builder()
                .name("rule-attck-mapping")
                .capabilityName("attck-mapping")
                .function((Map<String, Object> input) -> {
                    @SuppressWarnings("unchecked")
                    var alert = (Map<String, Object>) input.getOrDefault("alert", Map.of());
                    String alertRule = (String) alert.getOrDefault("rule", "");

                    @SuppressWarnings("unchecked")
                    var enrichment = (Map<String, Object>) input.getOrDefault("iocEnrichment", Map.of());
                    @SuppressWarnings("unchecked")
                    var iocs = (List<Map<String, Object>>) enrichment.getOrDefault("iocs", List.of());
                    var iocTypes = iocs.stream()
                            .map(m -> (String) m.get("type"))
                            .collect(Collectors.toList());

                    AttckMappingOutput mapping = AttckLookupTable.lookup(alertRule, iocTypes);
                    var enriched = mapping.techniques().stream()
                            .map(t -> enrichTechnique(t, enrichmentService))
                            .collect(Collectors.toList());
                    return WorkerResult.of(Map.of(
                            "techniques", enriched,
                            "primaryTactic", mapping.primaryTactic(),
                            "confidence", mapping.confidence(),
                            "narrative", mapping.narrative()));
                })
                .build();
    }

    private static Map<String, Object> enrichTechnique(
            AttckMappingOutput.TechniqueEntry entry,
            AttckEnrichmentService enrichmentService) {
        var groups = enrichmentService.getRelatedGroups(entry.technique()).stream()
                .map(AttckRelatedEntity::name).toList();
        var mitigations = enrichmentService.getMitigations(entry.technique()).stream()
                .map(e -> e.mitreId() + " — " + e.name()).toList();
        var subTechniques = enrichmentService.getSubTechniques(entry.technique()).stream()
                .map(e -> e.mitreId() + " — " + e.name()).toList();
        return Map.of(
                "technique", entry.technique(),
                "confidence", entry.confidence(),
                "evidence", entry.evidence(),
                "relatedGroups", groups,
                "mitigations", mitigations,
                "subTechniques", subTechniques);
    }
}
```

- [ ] **Step 6: Update SocInvestigationCaseDescriptor**

Add `AttckEnrichmentService` as a constructor parameter. Update the `workers()` method to pass it to `RuleAttckMappingWorker.create()`:

```java
// Add field
private final AttckEnrichmentService attckEnrichmentService;

// Update constructors
SocInvestigationCaseDescriptor() {
    this(null, null, null, null);
}

SocInvestigationCaseDescriptor(ChatModel llmModel,
                               SocCbrRetrieveService cbrRetrieveService,
                               ContainmentExecutor containmentExecutor,
                               AttckEnrichmentService attckEnrichmentService) {
    this.llmModel = llmModel;
    this.cbrRetrieveService = cbrRetrieveService;
    this.containmentExecutor = containmentExecutor;
    this.attckEnrichmentService = attckEnrichmentService;
}

// Update workers() — change the RuleAttckMappingWorker.create() call
RuleAttckMappingWorker.create(attckEnrichmentService),
```

- [ ] **Step 7: Update SocCaseHub to inject and pass AttckEnrichmentService**

```java
@Inject
AttckEnrichmentService attckEnrichmentService;

// In augment():
var descriptor = new SocInvestigationCaseDescriptor(
        null, cbrRetrieveService, containmentExecutor, attckEnrichmentService);
```

- [ ] **Step 8: Update SocInvestigationCaseDescriptorTest**

Update the constructor call in the test to pass `null` as the fourth parameter (or a mock):

```java
// Find and update the test constructor call — pass null for AttckEnrichmentService
new SocInvestigationCaseDescriptor(null, null, null, null)
```

- [ ] **Step 9: Run all tests to verify everything passes**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode test -pl app -am -Dsurefire.failIfNoSpecifiedTests=false`
Expected: PASS — all tests green (parser, enrichment, ingestion, worker, descriptor)

- [ ] **Step 10: Commit**

```bash
git add app/src/main/java/io/casehub/soc/ app/src/test/java/io/casehub/soc/
git commit -m "feat(#56): wire ATT&CK enrichment into investigation worker

Move AttckLookupTable to threatintel.attck package. RuleAttckMappingWorker
now enriches static mappings with graph context — related groups,
mitigations, and sub-techniques from MindMap. Wired through
SocInvestigationCaseDescriptor and SocCaseHub.

Refs #56"
```

- [ ] **Step 11: Run full build to confirm**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn --batch-mode install -pl app -am`
Expected: BUILD SUCCESS

---

## References

- [2026-09-15-attck-stix-ingestion-design.md] — design spec this plan implements
- [decisions.md] — D1 (app/ module), D2 (startup ingestion), D3 (keep both lookup sources)
- [app/src/main/java/io/casehub/soc/worker/RuleAttckMappingWorker.java:15] — existing worker factory
- [app/src/main/java/io/casehub/soc/engine/SocCaseHub.java:25] — augment() wiring point
- [app/src/main/java/io/casehub/soc/engine/SocInvestigationCaseDescriptor.java:31] — constructor and workers()
- [app/src/main/java/io/casehub/soc/worker/AttckLookupTable.java:7] — static lookup (moves to attck package)
- [api/src/main/java/io/casehub/soc/worker/contract/AttckMappingOutput.java:5] — worker output contract (stays in api/)
- [GitHub #56] — ATT&CK STIX ingestion issue
- [GitHub #51] — parent epic — RAG-powered investigation enrichment
