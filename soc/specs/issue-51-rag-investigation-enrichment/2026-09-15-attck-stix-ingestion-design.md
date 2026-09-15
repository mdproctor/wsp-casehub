# ATT&CK STIX Ingestion — MindMap Structural Index + RAG Prose Corpus

**Issue:** #56 — ATT&CK STIX ingestion — MindMap structural index + RAG prose corpus
**Parent:** Epic #51 — RAG-powered investigation enrichment
**Date:** 2026-09-15

---

## Problem

SOC's ATT&CK integration is a static lookup table (`AttckLookupTable`) that maps rule prefixes and IOC types to technique IDs. It returns a technique name and ID — nothing more. An analyst investigating a novel attack has no way to traverse "what else does this threat group use", "what mitigates this technique", or "show me the detection guidance for T1003". The structural relationships and prose descriptions that make ATT&CK valuable are locked in a JSON file nobody queries.

## Approach

Parse the MITRE ATT&CK Enterprise STIX 2.1 bundle at application startup and ingest into neocortex's two representations:

1. **MindMap** — techniques, groups, mitigations, malware, tools as nodes in a `mitre-attack` subgraph, distinguished by traits; `uses`, `mitigates`, `subtechnique-of` as edges. Structural queries traverse the graph.
2. **RAG corpus** — technique descriptions and detection guidance as prose chunks ingested via `EmbeddingIngestor`. Natural language queries retrieve relevant technique context.

Linked via `ChunkInput.metadata("mitreId", mitreId)` — a `RetrievedChunk` carries a MITRE ID in its metadata that can be resolved to a MindMap node for graph context.

The existing `AttckLookupTable` stays for fast deterministic mapping (Decision D3 — see `decisions.md`). `RuleAttckMappingWorker` enriches its output with MindMap graph context after the static lookup.

## Tenancy Model

ATT&CK is a public reference knowledge base — it must not be duplicated per tenant. The spec proposes a **reference tenant** convention:

```java
// AttckConstants.java — shared constants for the ATT&CK package
public final class AttckConstants {
    public static final String REFERENCE_TENANT = "__reference__";
    public static final String SUBGRAPH_NAME = "mitre-attack";
    public static final String CORPUS_NAME = "mitre-attack";
    private AttckConstants() {}
}
```

All ATT&CK MindMap data and RAG corpus data are stored under this tenant ID. Both `AttckIngestionService` and `AttckEnrichmentService` reference `AttckConstants.REFERENCE_TENANT` — no magic string duplication.

For RAG: `CorpusRef(REFERENCE_TENANT, CORPUS_NAME)`.

**Package:** `io.casehub.soc.threatintel.attck`

DECISION_NEEDED: The platform has no existing convention for shared/system tenant IDs. This proposal introduces one. Should `REFERENCE_TENANT` live in `casehub-platform-api` as a platform-wide constant, or remain SOC-local? If platform-wide, other applications with shared reference data (e.g., regulatory corpora) would use the same convention.

## STIX 2.1 Bundle

Source: `github.com/mitre-attack/attack-stix-data` — `enterprise-attack/enterprise-attack.json`

The bundle is a JSON array of STIX Domain Objects (SDOs). Each SDO has `type`, `id`, `name`, `description`, and type-specific properties.

### Relevant STIX types

| STIX type | Count (~) | What it represents |
|---|---|---|
| `attack-pattern` | ~600 | Techniques and sub-techniques |
| `intrusion-set` | ~140 | Threat groups (APT28, Lazarus, etc.) |
| `course-of-action` | ~400 | Mitigations |
| `malware` | ~450 | Malware families |
| `tool` | ~80 | Legitimate tools used by adversaries |
| `relationship` | ~15,000 | Typed edges between objects |
| `x-mitre-collection` | 1 | ATT&CK dataset metadata (version, modified date) |

### STIX object structure (attack-pattern example)

```json
{
  "type": "attack-pattern",
  "id": "attack-pattern--1234-...",
  "name": "OS Credential Dumping",
  "description": "Adversaries may attempt to dump credentials...",
  "external_references": [
    {"source_name": "mitre-attack", "external_id": "T1003"}
  ],
  "kill_chain_phases": [
    {"kill_chain_name": "mitre-attack", "phase_name": "credential-access"}
  ],
  "x_mitre_platforms": ["Windows", "Linux", "macOS"],
  "x_mitre_data_sources": ["Process: OS API Execution", ...],
  "x_mitre_detection": "Monitor for unexpected processes...",
  "x_mitre_version": "1.4",
  "x_mitre_is_subtechnique": false
}
```

### STIX relationship structure

```json
{
  "type": "relationship",
  "relationship_type": "uses",
  "source_ref": "intrusion-set--abc-...",
  "target_ref": "attack-pattern--1234-..."
}
```

### Filtering

Objects with `x_mitre_deprecated: true` or `revoked: true` are excluded during parsing. These represent retired techniques/groups that should not appear in enrichment results or RAG retrieval.

### Version Detection

The ATT&CK dataset version is extracted from the `x-mitre-collection` SDO's `x_mitre_version` field (e.g., `"16.1"`). This is distinct from STIX's `spec_version` (the STIX specification version) and individual SDO `x_mitre_version` fields (per-object versions). The collection's `x_mitre_version` is compared against the stored version on startup to determine whether re-ingestion is needed.

## New Components

### AttckStixParser

Pure Java parser for the STIX 2.1 JSON bundle. No external STIX library — the format is simple enough for Jackson.

```java
public class AttckStixParser {
    public AttckBundle parse(InputStream stixJson);
}
```

`AttckBundle` is a record holding lists of parsed objects:
```java
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

Each parsed type is a record extracting the fields relevant to MindMap/RAG:
```java
public record AttckTechnique(
    String stixId, String mitreId, String name, String description,
    String detection, List<String> tactics, List<String> platforms,
    List<String> dataSources, String version, boolean isSubtechnique
) {}

public record AttckGroup(
    String stixId, String mitreId, String name, String description,
    List<String> aliases
) {}

public record AttckMitigation(
    String stixId, String mitreId, String name, String description
) {}

public record AttckMalware(
    String stixId, String mitreId, String name, String description
) {}

public record AttckTool(
    String stixId, String mitreId, String name, String description
) {}

public record AttckRelationship(
    String stixId, String sourceRef, String targetRef, String relationshipType
) {}
```

Note: `AttckRelationship.relationshipType()` maps to the STIX `relationship_type` field (e.g., `"uses"`, `"mitigates"`), NOT the `type` field (which is always `"relationship"` for relationship SDOs).

The parser filters out deprecated and revoked objects during parsing. The `version` field on `AttckBundle` is extracted from the `x-mitre-collection` SDO.

**Package:** `io.casehub.soc.threatintel.attck`
**Module:** `app/`

### AttckIngestionService

`@Startup @ApplicationScoped` — runs at application boot. Checks if the `mitre-attack` subgraph exists in MindMap with the current bundle version. If missing or stale, parses the classpath STIX bundle and ingests.

```java
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
        // 1. Load STIX bundle from classpath
        var stixStream = getClass().getResourceAsStream("/threatintel/enterprise-attack.json");
        if (stixStream == null) {
            Log.warn("ATT&CK STIX bundle not found on classpath — skipping ingestion");
            return;
        }

        // 2. Version check — resolve stored version from subgraph root node.
        //    Handles crash recovery: any subgraph without a valid version stamp
        //    (null rootNodeId, missing root node, missing version property)
        //    is treated as incomplete and erased before re-ingestion.
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

        // 3. Parse STIX bundle
        AttckBundle bundle = new AttckStixParser().parse(stixStream);
        if (bundle.version().equals(storedVersion)) return;

        // 4. If re-ingestion: erase old subgraph (removes nodes, edges, aliases)
        if (existing != null) {
            mindMapStore.eraseSubgraph(existing.id(), REFERENCE_TENANT);
        }

        // 5. Corpus cleanup — required before RAG re-ingestion to prevent
        //    duplicates. If cleanup fails, RAG ingestion is skipped entirely
        //    so old+new chunks never coexist.
        CorpusRef corpus = new CorpusRef(REFERENCE_TENANT, CORPUS_NAME);
        boolean corpusCleanupFailed = false;
        try {
            embeddingIngestor.deleteCorpus(corpus);
        } catch (Exception e) {
            Log.warn("deleteCorpus failed — RAG re-ingestion will be skipped to prevent duplicates", e);
            corpusCleanupFailed = true;
        }

        // 6. Register vocabulary (idempotent)
        mindMapStore.registerVocabulary(MindMapVocabulary.builder()
            .edgeType("uses").edgeType("mitigates").edgeType("subtechnique-of")
            .build());

        // 7. Create subgraph (rootNodeId null initially — set in step 12)
        String subgraphId = mindMapStore.createSubgraph(
            new SubgraphInput(SUBGRAPH_NAME, SUBGRAPH_NAME, null),
            REFERENCE_TENANT);

        // 8. Create nodes — capture stixId→storeId map for edge creation
        Map<String, String> stixToStoreId = new HashMap<>();
        for (var t : bundle.techniques()) {
            String storeId = mindMapStore.addNode(
                buildTechniqueInput(t, subgraphId), REFERENCE_TENANT);
            mindMapStore.addAlias(storeId, t.mitreId(), REFERENCE_TENANT);
            stixToStoreId.put(t.stixId(), storeId);
        }
        // groups, mitigations, malware, tools created similarly

        // 9. Create edges — log dropped count for observability
        int droppedSrcMissing = 0, droppedTgtMissing = 0;
        for (var rel : bundle.relationships()) {
            String src = stixToStoreId.get(rel.sourceRef());
            String tgt = stixToStoreId.get(rel.targetRef());
            if (src != null && tgt != null) {
                mindMapStore.addEdge(
                    EdgeInput.of(src, tgt, rel.relationshipType()), REFERENCE_TENANT);
            } else {
                if (src == null) droppedSrcMissing++;
                if (tgt == null) droppedTgtMissing++;
            }
        }
        if (droppedSrcMissing + droppedTgtMissing > 0) {
            Log.infof("ATT&CK edges dropped: %d (source missing: %d, target missing: %d)",
                droppedSrcMissing + droppedTgtMissing, droppedSrcMissing, droppedTgtMissing);
        }

        // 10. RAG ingestion — skipped if corpus cleanup failed (step 5) to
        //     prevent permanent duplicates. MindMap works without RAG.
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

        // 11. Create root node with version, set as subgraph root.
        //     MUST be last data-writing step — version stamp = ingestion complete.
        //     Proceeds regardless of RAG outcome — MindMap is the primary data.
        //     Crash before this point → next startup sees null rootNodeId or
        //     missing version → treats as STALE → erase + re-ingest.
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
}
```

All constants (`REFERENCE_TENANT`, `SUBGRAPH_NAME`, `CORPUS_NAME`) reference `AttckConstants` via static import.

**Alias registration:** During node creation (step 7), each node's MITRE ID is registered as an alias via `addAlias(storeId, mitreId, REFERENCE_TENANT)`. This enables `AttckEnrichmentService` to resolve MITRE IDs directly through the MindMap SPI's `resolveNode()` — no custom cache or CDI event coordination is needed. On re-ingestion, `eraseSubgraph()` removes all nodes and their associated aliases, so the re-ingestion flow re-registers fresh aliases.

**Vocabulary registration:** Registers a `MindMapVocabulary` defining the three edge types. `registerVocabulary()` is idempotent for identical definitions — safe to call on every ingestion:
```java
mindMapStore.registerVocabulary(MindMapVocabulary.builder()
    .edgeType("uses")
    .edgeType("mitigates")
    .edgeType("subtechnique-of")
    .build());
```

**Version control and crash recovery:** A dedicated root node (name `"mitre-attack-root"`, trait `"metadata"`) is created as the **last data-writing step** of ingestion, storing `attck-version` as a property. The subgraph's `rootNodeId` is set to this node via `updateSubgraph()`. This ordering provides idempotent crash recovery without transactions:

1. `listSubgraphs(REFERENCE_TENANT)` → find `"mitre-attack"`
2. If not found → **FRESH_INGEST** (no erase needed)
3. If found → check `sg.rootNodeId()`
4. If `rootNodeId` is null → **STALE** (crash after `createSubgraph` but before root node creation)
5. Else → `getNode(rootNodeId, REFERENCE_TENANT)`
6. If node is null → **STALE** (root node lost — corrupted state)
7. Else → `node.property("attck-version")`
8. If missing or differs from bundle version → **STALE** (version mismatch)
9. If matches → **SKIP** (data is current)

STALE path: erase old subgraph + delete corpus → re-ingest fresh. The version stamp as the last step is the critical invariant: any subgraph without a valid version stamp is, by definition, incomplete and must be cleaned up.

**Node ID mapping:** `MindMapStore.addNode()` returns a store-generated UUID. During ingestion, the service maintains a transient `Map<String, String>` mapping STIX IDs to store-generated UUIDs. This map is used when creating edges — `EdgeInput.of(sourceStoreId, targetStoreId, edgeType)`.

**Package:** `io.casehub.soc.threatintel.attck`
**Module:** `app/`

### STIX → MindMap Mapping

All ATT&CK entities are nodes in a single `mitre-attack` subgraph. Entity types are distinguished by **traits** — `MindMapNode.traits()` provides categorical labels. The subgraph type `mitre-attack` applies to all nodes (per the MindMap `type() = subgraphType()` contract).

| STIX type | Trait | Properties |
|---|---|---|
| `attack-pattern` | `attack-technique` | `mitreId`, `name`, `tactics` (comma-sep, ALL kill chain phases), `platforms` (comma-sep), `dataSources` (comma-sep), `version`, `isSubtechnique` |
| `intrusion-set` | `threat-group` | `mitreId`, `name`, `aliases` (comma-sep) |
| `course-of-action` | `mitigation` | `mitreId`, `name` |
| `malware` | `malware` | `mitreId`, `name` |
| `tool` | `attack-tool` | `mitreId`, `name` |

Node names use the ATT&CK name (e.g., "OS Credential Dumping"). Each node also carries a `NodeRef(scheme="mitre-attack", id=mitreId)` for cross-referencing.

Example node creation:
```java
NodeInput input = NodeInput.of("OS Credential Dumping", subgraphId)
    .withTraits(Set.of("attack-technique"))
    .withRefs(Set.of(new NodeRef("mitre-attack", "T1003", null)))
    .withProperties(Map.of(
        "mitreId", "T1003",
        "tactics", "credential-access",
        "platforms", "Windows,Linux,macOS",
        "dataSources", "Process: OS API Execution",
        "version", "1.4",
        "isSubtechnique", "false"
    ));
String storeId = mindMapStore.addNode(input, REFERENCE_TENANT);
mindMapStore.addAlias(storeId, "T1003", REFERENCE_TENANT);
stixToStoreId.put("attack-pattern--1234-...", storeId);
```

### STIX Relationships → MindMap Edges

| STIX `relationship_type` | Source type | Target type | MindMap edge type |
|---|---|---|---|
| `uses` | `intrusion-set` | `attack-pattern` | `uses` |
| `uses` | `malware` | `attack-pattern` | `uses` |
| `uses` | `tool` | `attack-pattern` | `uses` |
| `uses` | `intrusion-set` | `malware` | `uses` |
| `uses` | `intrusion-set` | `tool` | `uses` |
| `mitigates` | `course-of-action` | `attack-pattern` | `mitigates` |
| `subtechnique-of` | `attack-pattern` | `attack-pattern` | `subtechnique-of` |

Other relationship types (e.g., `revoked-by`, `attributed-to`) are skipped for now — they can be added later without schema changes.

Edge creation uses the STIX ID → store ID map:
```java
String sourceStoreId = stixToStoreId.get(relationship.sourceRef());
String targetStoreId = stixToStoreId.get(relationship.targetRef());
if (sourceStoreId != null && targetStoreId != null) {
    mindMapStore.addEdge(
        EdgeInput.of(sourceStoreId, targetStoreId, relationship.relationshipType()),
        REFERENCE_TENANT
    );
}
```

### Prose → RAG Corpus

Each technique's `description` and `x_mitre_detection` fields are concatenated and ingested as a chunk via `EmbeddingIngestor`. The `x_mitre_detection` field is optional — techniques without detection guidance omit it:

```java
CorpusRef corpus = new CorpusRef(REFERENCE_TENANT, "mitre-attack");

// Batch all technique chunks in a single ingest() call
List<ChunkInput> techniqueChunks = techniques.stream()
    .map(t -> {
        String content = t.detection() != null
            ? t.description() + "\n\n" + t.detection()
            : t.description();
        return new ChunkInput(
            content,                              // content (first parameter)
            t.mitreId(),                          // sourceDocumentId (second parameter)
            Map.of(
                "mitreId", t.mitreId(),
                "name", t.name(),
                "tactics", String.join(",", t.tactics()),
                "type", "attack-technique"
            )
        );
    })
    .toList();
embeddingIngestor.ingest(corpus, techniqueChunks);
```

Group and mitigation descriptions are batched and ingested similarly, each batch in a single `ingest()` call with respective type metadata:

```java
// Group chunks — description + aliases for semantic matching
List<ChunkInput> groupChunks = groups.stream()
    .map(g -> new ChunkInput(
        g.description(),
        g.mitreId(),
        Map.of(
            "mitreId", g.mitreId(),
            "name", g.name(),
            "type", "threat-group"
        )
    ))
    .toList();
embeddingIngestor.ingest(corpus, groupChunks);

// Mitigation chunks
List<ChunkInput> mitigationChunks = mitigations.stream()
    .map(m -> new ChunkInput(
        m.description(),
        m.mitreId(),
        Map.of(
            "mitreId", m.mitreId(),
            "name", m.name(),
            "type", "mitigation"
        )
    ))
    .toList();
embeddingIngestor.ingest(corpus, mitigationChunks);

// Malware chunks
List<ChunkInput> malwareChunks = malware.stream()
    .map(m -> new ChunkInput(
        m.description(),
        m.mitreId(),
        Map.of(
            "mitreId", m.mitreId(),
            "name", m.name(),
            "type", "malware"
        )
    ))
    .toList();
embeddingIngestor.ingest(corpus, malwareChunks);

// Tool chunks
List<ChunkInput> toolChunks = tools.stream()
    .map(t -> new ChunkInput(
        t.description(),
        t.mitreId(),
        Map.of(
            "mitreId", t.mitreId(),
            "name", t.name(),
            "type", "attack-tool"
        )
    ))
    .toList();
embeddingIngestor.ingest(corpus, toolChunks);
```

The `type` metadata key is the discriminator for #57's retrieval worker — it can filter results by entity type (e.g., only techniques, techniques + groups, or malware). This enables queries like "credential dumping against Active Directory" to surface technique descriptions, group TTPs, and associated malware/tool information.

**Corpus scoping:** All ATT&CK prose lives in `CorpusRef(REFERENCE_TENANT, "mitre-attack")`. Internal knowledge (post-mortems, #58) uses a separate corpus. Retrieval can target one or both.

**Metadata for #57 (RAG retrieval worker):** The metadata fields — `mitreId`, `name`, `tactics`, `type` — are sufficient for #57's retrieval patterns. `RetrievedChunk.metadata()` exposes these fields, allowing the retrieval worker to filter by tactic or resolve the MITRE ID to a MindMap node. If #57 needs additional metadata (e.g., `platforms`, `dataSources`), it can be added to the chunk metadata without schema changes.

### AttckEnrichmentService

Provides MindMap graph queries for investigation workers. Wraps `MindMapStore.neighbors()` with SOC-specific convenience methods that handle edge→node resolution, direction filtering, and trait-based type filtering.

MITRE ID → node resolution uses the MindMap alias system: during ingestion, each node's MITRE ID is registered as an alias via `addAlias()`. The enrichment service resolves MITRE IDs via `resolveNode(mitreId, null, REFERENCE_TENANT)` — no custom in-memory cache, CDI event coordination, volatile fields, or synchronization needed. Edge targets are resolved via `getNode(storeId, REFERENCE_TENANT)`, which is a direct primary-key lookup. On re-ingestion, `eraseSubgraph()` removes all nodes and their aliases; subsequent `resolveNode()` calls naturally return null until re-ingestion completes (graceful degradation).

```java
@ApplicationScoped
public class AttckEnrichmentService {

    @Inject MindMapStore mindMapStore;

    public List<AttckRelatedEntity> getRelatedGroups(String mitreId) {
        MindMapNode node = mindMapStore.resolveNode(mitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "uses", REFERENCE_TENANT).stream()
            .filter(edge -> edge.targetNodeId().equals(node.id())) // INBOUND
            .map(edge -> mindMapStore.getNode(edge.sourceNodeId(), REFERENCE_TENANT))
            .filter(n -> n != null && n.traits().contains("threat-group"))
            .map(AttckRelatedEntity::from)
            .toList();
    }

    public List<AttckRelatedEntity> getMitigations(String mitreId) {
        MindMapNode node = mindMapStore.resolveNode(mitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "mitigates", REFERENCE_TENANT).stream()
            .filter(edge -> edge.targetNodeId().equals(node.id())) // INBOUND
            .map(edge -> mindMapStore.getNode(edge.sourceNodeId(), REFERENCE_TENANT))
            .filter(n -> n != null && n.traits().contains("mitigation"))
            .map(AttckRelatedEntity::from)
            .toList();
    }

    public List<AttckRelatedEntity> getSubTechniques(String mitreId) {
        MindMapNode node = mindMapStore.resolveNode(mitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "subtechnique-of", REFERENCE_TENANT).stream()
            .filter(edge -> edge.targetNodeId().equals(node.id())) // INBOUND
            .map(edge -> mindMapStore.getNode(edge.sourceNodeId(), REFERENCE_TENANT))
            .filter(Objects::nonNull)
            .map(AttckRelatedEntity::from)
            .toList();
    }

    public List<AttckRelatedEntity> getGroupTechniques(String groupMitreId) {
        MindMapNode node = mindMapStore.resolveNode(groupMitreId, null, REFERENCE_TENANT);
        if (node == null) return List.of();
        return mindMapStore.neighbors(node.id(), "uses", REFERENCE_TENANT).stream()
            .filter(edge -> edge.sourceNodeId().equals(node.id())) // OUTBOUND
            .map(edge -> mindMapStore.getNode(edge.targetNodeId(), REFERENCE_TENANT))
            .filter(n -> n != null && n.traits().contains("attack-technique"))
            .map(AttckRelatedEntity::from)
            .toList();
    }
}
```

All `REFERENCE_TENANT` references use `AttckConstants.REFERENCE_TENANT` via static import.

```java
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
            trait
        );
    }
}
```

**Single-trait invariant:** Each ATT&CK node is created with exactly one trait (see §STIX → MindMap Mapping). `ATTCK_TRAITS` provides a deterministic filter — if a node ever acquires additional non-ATT&CK traits (e.g., from cross-subgraph edges), the filter selects the ATT&CK-specific one.

**Deferred scope — malware/tool enrichment:** MindMap nodes for `malware` and `tool` STIX types are ingested with `uses` edges (malware→technique, tool→technique, group→malware, group→tool), but `AttckEnrichmentService` does not yet expose methods to query them (e.g., `getRelatedMalware(techniqueId)`, `getRelatedTools(techniqueId)`, `getGroupMalware(groupId)`). This is intentional:
1. **Graph integrity** — malware/tool nodes must exist for edge creation to succeed; skipping them would silently drop ~500 relationships.
2. **RAG availability** — malware/tool descriptions are ingested into the RAG corpus (§Prose → RAG Corpus), so #57's retrieval worker can surface them via semantic search without dedicated enrichment API methods.
3. **Future extension** — adding enrichment methods is additive and requires no schema change.
Tracked as deferred work in #56's comments.

**Package:** `io.casehub.soc.threatintel.attck`

### Package consolidation

`AttckLookupTable` currently lives in `io.casehub.soc.worker` alongside the worker that uses it. It is ATT&CK domain knowledge, not worker infrastructure — move it to `io.casehub.soc.threatintel.attck` to consolidate all ATT&CK domain logic in one package. The worker (`RuleAttckMappingWorker`) stays in `io.casehub.soc.worker`; the dependency direction (worker → domain) is correct. `AttckMappingOutput` stays in `api/.../worker/contract/` — it is a worker output contract, not ATT&CK domain knowledge.

### RuleAttckMappingWorker Enhancement

The existing worker's factory method is updated to accept `AttckEnrichmentService` as a parameter:

```java
public static Worker create(AttckEnrichmentService enrichmentService) {
    return Worker.builder()
        .name("rule-attck-mapping")
        .capabilityName("attck-mapping")
        .function((Map<String, Object> input) -> {
            // ... existing AttckLookupTable logic ...
            AttckMappingOutput mapping = AttckLookupTable.lookup(alertRule, iocTypes);

            // Enrich with MindMap context (optional — graceful if service unavailable)
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
```

**Integration via established worker registration pattern:** Workers are plain Java objects assembled in `SocInvestigationCaseDescriptor`, not CDI beans (see `docs/guides/contributor-guide.md`). `SocCaseHub` injects the enrichment service and passes it through:

```java
// SocCaseHub.java — add @Inject for AttckEnrichmentService
@Inject AttckEnrichmentService attckEnrichmentService;

@Override
protected void augment(CaseDefinition definition) {
    var descriptor = new SocInvestigationCaseDescriptor(
        null, cbrRetrieveService, containmentExecutor, attckEnrichmentService);
    definition.getWorkers().addAll(descriptor.workers());
    // ...
}

// SocInvestigationCaseDescriptor — add constructor parameter
SocInvestigationCaseDescriptor(ChatModel llmModel,
                               SocCbrRetrieveService cbrRetrieveService,
                               ContainmentExecutor containmentExecutor,
                               AttckEnrichmentService attckEnrichmentService) {
    // ...
}

// workers() — pass enrichmentService to factory method
RuleAttckMappingWorker.create(attckEnrichmentService),
```

**Current output:**
```json
{"technique": "T1003", "tactic": "credential-access", "confidence": 0.8}
```

**Enriched output:**
```json
{
  "technique": "T1003",
  "tactic": "credential-access",
  "confidence": 0.8,
  "relatedGroups": ["APT28", "APT29", "Lazarus Group"],
  "mitigations": ["M1026 — Privileged Account Management", "M1043 — Credential Access Protection"],
  "subTechniques": ["T1003.001 — LSASS Memory", "T1003.002 — Security Account Manager"]
}
```

The `enrichTechnique` helper transforms each static `TechniqueEntry` into a map with graph context:

```java
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
```

Enrichment is optional — if `AttckEnrichmentService` resolves no node (MindMap not populated or alias not found), the enrichment methods return empty lists and the worker output is equivalent to the current static output.

## Dependencies

### Maven dependencies (new)

```xml
<!-- MindMap API + runtime impl -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-core</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-sqlite</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-inmem</artifactId>
    <scope>test</scope>
</dependency>

<!-- RAG API + in-memory test impl -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-rag-api</artifactId>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-rag</artifactId>
    <scope>runtime</scope>
</dependency>
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-rag-testing</artifactId>
    <scope>test</scope>
</dependency>

<!-- MindMap testing (contract test base) -->
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-mindmap-testing</artifactId>
    <scope>test</scope>
</dependency>
```

### Classpath resource

The STIX bundle is placed at:
```
app/src/main/resources/threatintel/enterprise-attack.json
```

This adds ~15MB to the application JAR. Acceptable for the pre-release phase. In production, the bundle path could be made configurable for external loading.

### application.properties

```properties
# ATT&CK ingestion — disable to skip startup ingestion (e.g., in tests)
casehub.soc.attck.enabled=true
```

The `enabled` flag is consumed by `AttckIngestionService` via `@ConfigProperty` (see §AttckIngestionService). The corpus name and subgraph name are constants in `AttckConstants` — not configurable, because multiple components (`AttckIngestionService`, `AttckEnrichmentService`, and #57's retrieval worker) must agree on the same values.

## Testing Strategy

### Unit tests

- **AttckStixParserTest** — parse a minimal STIX bundle (~10 techniques, ~5 groups, ~20 relationships), verify all object types and relationships are extracted; verify deprecated/revoked objects are filtered
- **AttckEnrichmentServiceTest** — mock MindMapStore, verify `resolveNode()` and `getNode()` delegation, graph traversal queries for groups, mitigations, sub-techniques; verify trait filtering and null-node handling

### Integration test

- **AttckIngestionIntegrationTest** (`@QuarkusTest` with in-memory MindMap + RAG) — ingest a small STIX subset, verify:
  - MindMap nodes exist with correct traits and properties
  - MindMap edges exist for relationships with correct direction
  - RAG retrieval returns technique, group, mitigation, malware, and tool descriptions for natural language queries
  - `AttckEnrichmentService` returns correct graph traversal results
  - Version check prevents re-ingestion on second startup
  - Vocabulary registration succeeds (edge types are `REGISTERED` tier)
  - **Alias registration:** Verify MITRE IDs are registered as aliases during ingestion — `resolveNode(mitreId, null, REFERENCE_TENANT)` returns the correct node
  - **Re-ingestion flow:** Ingest V1 → verify enrichment returns data → simulate restart with V2 bundle → verify old data erased (including aliases), new data ingested with fresh aliases, enrichment returns V2 data
  - **Graceful degradation during erase:** Call enrichment after erase but before re-ingestion completes → verify empty results (not exception), since `resolveNode()` returns null for erased aliases
  - **Root node version storage:** Verify subgraph root node exists with `attck-version` property matching bundle version; verify `updateSubgraph()` sets rootNodeId correctly
  - **Crash recovery — incomplete ingestion:** Create subgraph with null rootNodeId (simulating crash before root node creation) → restart ingestion → verify subgraph is erased and re-ingested cleanly
  - **Crash recovery — orphan corpus:** Ingest RAG data, then simulate crash before version stamp → restart → verify `deleteCorpus` runs before re-ingestion, no duplicate chunks
  - **Error boundary:** Force an exception during ingestion (e.g., read-only MindMapStore mock) → verify application starts, no exception propagates to Quarkus startup, enrichment returns empty
  - **Edge drop logging:** Ingest a bundle with relationships referencing non-extracted STIX types → verify dropped edge count is logged at INFO
  - **RAG failure isolation:** Mock `EmbeddingIngestor` to throw on `ingest()` → verify MindMap nodes/edges are created, version stamp is written, enrichment service returns graph results, and application starts normally
  - **deleteCorpus failure prevents RAG duplicates:** Mock `EmbeddingIngestor.deleteCorpus()` to throw → verify RAG `ingest()` is NOT called (step 10 skipped), MindMap nodes/edges are created, version stamp is written
  - **NoOpMindMapStore capability guard:** Configure `NoOpMindMapStore` (no capabilities) → verify ingestion is skipped with INFO log, no MindMap/RAG operations attempted, no wasted parse
  - **Malware/tool RAG ingestion:** Verify malware and tool descriptions are ingested as RAG chunks with correct `type` metadata (`"malware"`, `"attack-tool"`); verify retrieval returns these chunks for semantic queries

## Failure Modes

| Failure | Behavior |
|---|---|
| STIX bundle missing from classpath | Log warning, skip ingestion. AttckLookupTable still works. |
| STIX bundle malformed | Log error with parse location, skip ingestion. |
| MindMap store unavailable at startup | Log error, retry once after 5s. If still unavailable, skip. |
| RAG ingestor unavailable | Step 5 (`deleteCorpus`) failure sets `corpusCleanupFailed` flag — step 10 is skipped entirely to prevent permanent duplicates (stale RAG data remains). Step 10 failure is independently caught — MindMap ingestion and the version stamp proceed. Log warning, ingest MindMap only. Graph traversals work; prose retrieval unavailable or stale. Recovery: restart with RAG available and force re-ingest (bump bundle version or delete root node). |
| Version mismatch (re-ingestion) | Erase old subgraph + corpus (erasing also removes aliases), re-ingest fresh. Brief unavailability window where enrichment returns empty — `resolveNode()` returns null for erased aliases. MindMap unavailability: ~2-5s (steps 7-9). RAG corpus unavailability: ~15-55s (step 5 deletes corpus through step 10 completion). #57's retrieval worker returns empty results during this window. No cache rebuild needed — aliases are re-registered during ingestion. |
| Crash during ingestion | On restart: the version-check algorithm (§Version control and crash recovery) detects any subgraph without a valid version stamp — null `rootNodeId`, missing root node, or missing `attck-version` property — and treats it as incomplete. Stale subgraph is erased, corpus cleanup is attempted before RAG ingestion. If corpus cleanup fails, RAG ingestion is skipped (prevents duplicates); MindMap is still re-ingested and version-stamped. No duplicate nodes. The version stamp as the last data-writing step is the critical invariant. |
| Any uncaught exception in `onStartup` | Caught by top-level try-catch. Logged as error. Application starts normally — static `AttckLookupTable` still works. Enrichment service returns empty results (graceful degradation). |

## Startup Timing

First boot (cold — no ATT&CK data): ~1,670 nodes + ~15,000 edges + ~1,670 RAG chunks. Expected time with in-memory MindMap: 2-5s for MindMap writes, 15-50s for RAG embedding (depends on embedding model and hardware). Total: 20-55s added to first startup. Subsequent starts with matching version: ~2-5s (full bundle parse + version check, no ingestion).

The synchronous startup approach ensures ATT&CK data is available before any investigation worker fires. The enrichment service has no local state to build — it delegates directly to `MindMapStore.resolveNode()` and `getNode()`. If the application receives enrichment requests before ingestion completes, `resolveNode()` returns null and the enrichment methods return empty lists (graceful degradation, identical to the "MindMap not populated" fallback).

**Deployment notes:**
- **Minimum heap:** Cold-start ingestion holds the full STIX bundle (~15MB JSON), parsed records (~17,000 objects), and the stixToStoreId map in memory simultaneously. Peak ingestion memory is ~50-80MB. Minimum heap for cold start: `-Xmx256m` recommended.
- **Kubernetes probes:** Set `initialDelaySeconds` for liveness and readiness probes to at least 75s to account for cold-start ingestion (20-55s) plus normal Quarkus startup time. Without this, the liveness probe may kill the pod before ingestion completes, causing a restart loop.

## References

- `AttckLookupTable.java` — existing static ATT&CK lookup (stays, enriched via Decision D3)
- `RuleAttckMappingWorker.java` — existing worker (enhanced with graph enrichment)
- `MindMapStore.java` — neocortex graph SPI (nodes, edges, subgraphs, traversals)
- `MindMapNode.traits()` — categorical labels used for entity type filtering
- `NodeRef` — cross-reference identity (scheme="mitre-attack", id=mitreId)
- `MindMapStore.addAlias()` / `resolveNode()` — alias registration and lookup for MITRE ID resolution
- `MindMapStore.updateSubgraph()` — sets subgraph root node (for version storage)
- `MindMapVocabulary` — edge type registration for validation tier
- `EmbeddingIngestor.java` — neocortex RAG ingestion SPI
- `RetrievedChunk.metadata()` — RAG result metadata for MITRE ID linkage
- `CaseRetriever.java` — neocortex RAG retrieval SPI (used by #57, not this issue)
- `SocCbrRetrieveService.java` — existing CBR retrieval (parallel path, not modified)
- `github.com/mitre-attack/attack-stix-data` — ATT&CK STIX 2.1 source
- STIX 2.1 specification — object model for attack-pattern, intrusion-set, relationship
