## D1: mindmap-api gains platform-api dependency

**Choice:** Add `casehub-platform-api` as a compile dependency to `mindmap-api`
**Alternatives:**
- Keep String, convert at boundary — type safety only above the SPI, callers pass `.value()` manually
- Move typed ID to cognitive-api — parallel type that must stay in sync with PrincipalId
**Rationale:** PrincipalId is a simple record with zero transitive deps. Adding `casehub-platform-api` brings the full module onto mindmap-api's classpath (Path, ActorId, ParticipantId, and any future additions), but mindmap-api is casehub-only (not Hortora-eligible) and memory-api already depends on platform-api for `io.casehub.platform.api.path.Path` in CbrQuery. The practical cost is minimal. Consumers benefit from type safety at the SPI boundary — new fields should use the platform's typed identity model.
**Trade-offs:** mindmap-api is no longer platform-independent. Any project using mindmap-api without the platform would need to provide platform-api on the classpath.
**Sources:** NodeInput.java:22, EdgeInput.java:21, PrincipalId.java (platform-api), memory-api/pom.xml (existing platform-api dependency)
**Exploration:** quick
**Status:** revised (R1-10: rationale updated to acknowledge full module dependency)

## D2: MindMap gets principalId + sharedWith on nodes (full visibility parity with Memory)

**Choice:** Add `principalId` (PrincipalId, nullable) and `sharedWith` (Set<String>, nullable) as stored fields on `MindMapNode`, matching Memory's domain visibility model exactly
**Alternatives:**
- No MindMap visibility model — leave nodes visible to all principals in tenant
- principalId only, no sharedWith — simpler but no selective sharing
**Rationale:** MindMap IS memory (graph-structured). Tenant is infrastructure plumbing for physical separation, not a domain visibility model. The domain concern — "some knowledge is for a principal, some for a group" — applies equally to graph and text memory. Overlay nodes (per-agent PAD) are inherently private data that should not be visible to other agents. `sharedWith` uses `Set<String>` (PrincipalId values) because PrincipalVisibility in cognitive-api operates on String parameters (zero-deps constraint) — using `Set<PrincipalId>` would require conversion at every visibility check for no semantic benefit, since format validity is enforced at the input boundary via NodeInput construction.
**Trade-offs:** All MindMap backends must store and filter by these fields. Existing nodes get null principalId (public by default — backward compatible).
**Sources:** MemoryVisibility.java, MindMapNode.java, Memory.java:19, remove-memory-space-modules-design.md
**Exploration:** deep-analysis
**Status:** captured

## D3: MemoryVisibility moves to cognitive-api as PrincipalVisibility

**Choice:** Move the visibility predicate from `memory-api` to `cognitive-api` and rename to `PrincipalVisibility`. Drop the `Memory`-typed convenience overload — callers destructure at the call site.
**Alternatives:**
- Keep in memory-api — cognitive-index can reach it transitively, but name is misleading for MindMap use
- Accept PrincipalId parameters — requires cognitive-api to depend on platform-api, violating its zero-deps constraint
**Rationale:** The predicate now applies across MindMap, Memory, and CBR. cognitive-api is the zero-deps cross-cutting shared layer. The name "MemoryVisibility" is misleading when applied to MindMap nodes. The Memory-typed overload `isVisible(String, Memory)` was introduced in #277 with no external consumers — a deprecated forwarding class would be a backward-compatibility shim for an API with zero adoption. Callers with a Memory in hand call `PrincipalVisibility.isVisible(callerPrincipal, memory.principalId(), memory.sharedWith())`. Parameters remain `String`-typed because cognitive-api must stay zero-deps (no platform-api).
**Trade-offs:** Existing imports in memory-api consumers must update. Callers with PrincipalId call `.value()` before passing to PrincipalVisibility.
**Sources:** MemoryVisibility.java, cognitive-api/pom.xml (zero compile deps)
**Exploration:** quick
**Depends on:** D2 (MindMap needs the same visibility model)
**Status:** revised (R1-05: drop convenience overload; R1-06: String params retained for zero-deps constraint)

## D4: Query-level principal filtering in MindMap (not decorator)

**Choice:** Add `PrincipalId callerPrincipal` to `MindMapQuery`. Backends (inmem, SQLite) filter at query time. A contract test in `MindMapStoreContractTest` structurally enforces that every backend respects the field — stores a node with principalId "agent:alice", queries with callerPrincipal "agent:bob" (without sharedWith), and asserts the node is NOT returned.
**Alternatives:**
- Decorator-level filtering — single implementation but over-fetches and breaks pagination/limit contract
- Both (query hint + decorator fallback) — defense in depth but premature for 2 backends
**Rationale:** The limit interaction problem with decorators is real — post-query filtering breaks the contract that `limit=10` returns 10 results. With only 2 backends, implementation cost is trivial. Follows the same architectural pattern as CbrQuery.callerPrincipalId (query-level principal filtering), though the type differs: PrincipalId here vs String on CbrQuery (the type migration of existing String fields is a separate concern). The contract test structurally prevents any new backend from silently leaking data — any backend that ignores the field will fail CI.
**Trade-offs:** New MindMap backends must implement principal filtering and pass the contract test.
**Sources:** MindMapQuery.java, CbrQuery.java:26, InMemoryMindMapStore, SqliteMindMapStore, MindMapStoreContractTest.java (72 existing contract tests)
**Exploration:** quick
**Depends on:** D2 (principalId must be stored on nodes for backends to filter)
**Status:** revised (R1-03: consistency claim clarified as pattern not type; R1-04: contract test added)

## D5: #254 applies uniform visibility filtering across all stores in TemporalIndex

**Choice:** TemporalQuery gains a nullable `PrincipalId callerPrincipal`. TemporalIndex threads it to all three query types: MindMapQuery (as PrincipalId), MemoryQuery.callerPrincipalId (via `.value()`), and CbrQuery.callerPrincipalId (via `.value()`). All three stores filter at query time — no post-query filtering needed.
**Alternatives:**
- Memory/CBR only, no MindMap — incomplete, inconsistent with D2
- Defer #254 entirely — leaves TemporalIndex unaware of principal visibility
**Rationale:** With D2 giving MindMap the same visibility model as Memory, TemporalIndex can apply principal filtering uniformly. MemoryQuery already has a `callerPrincipalId` field (MemoryQuery.java:16) and SqliteMemoryStore already filters at query time (SqliteMemoryStore.java:397). CbrQuery similarly has `callerPrincipalId`. TemporalIndex simply passes `callerPrincipal.value()` through to each store's query builder — no post-query filtering, no PrincipalVisibility invocation needed at the TemporalIndex level.
**Trade-offs:** Type conversion (`.value()`) needed at TemporalIndex when bridging PrincipalId to String-typed query fields. This conversion disappears when Memory and CBR migrate their query types to PrincipalId.
**Sources:** TemporalIndex.java, TemporalQuery.java, MemoryQuery.java:16 (callerPrincipalId field exists), SqliteMemoryStore.java:397 (query-time filtering), CbrQuery.java:26
**Exploration:** deep-analysis
**Depends on:** D2, D3, D4
**Status:** revised (R1-09: corrected stale rationale — MemoryQuery already has callerPrincipalId)

## D6: Edges inherit visibility from their endpoint nodes

**Choice:** An edge is visible to a caller if and only if both its source and target nodes are visible to that caller. No separate principalId/sharedWith fields on MindMapEdge for visibility purposes.
**Alternatives:**
- Independent edge visibility — edges have their own principalId + sharedWith, visible even when endpoints are not
- Edges always visible — privacy is node-level only (information leak: edge existence reveals private node existence)
**Rationale:** In a knowledge graph, edges represent relationships between nodes. If a node is private to principal A, and an edge connects that node to a shared node, the edge reveals that the private node exists. An adversary querying `neighbors()` or `bridgeEdges()` could discover private nodes through their edge connections. Edge-level independent visibility adds unnecessary complexity — relationships don't have independent ownership from their endpoints in this domain. EdgeInput.principalId continues to serve its existing purpose (per-principal rule scoping in DerivedEdgeDecorator), which is orthogonal to visibility filtering.
**Trade-offs:** `neighbors()` and `bridgeEdges()` will need visibility enforcement — either a callerPrincipal parameter or endpoint visibility checks. MindMapStore interface changes required.
**Sources:** MindMapEdge.java, MindMapStore.java (neighbors, bridgeEdges), DerivedEdgeDecorator (EdgeInput.principalId for rule scoping)
**Exploration:** quick (surfaced by review)
**Depends on:** D2, D4
**Status:** captured

## D7: MindMapNode gains principalId() and sharedWith() — intentional interface break

**Choice:** Add `PrincipalId principalId()` and `Set<String> sharedWith()` to the MindMapNode interface. All implementations must be updated.
**Alternatives:**
- Default methods returning null — avoids compile break but hides the migration requirement
- Separate interface (VisibleMindMapNode extends MindMapNode) — adds type complexity for no architectural benefit
**Rationale:** This is an intentional breaking change. The breakage forces every MindMapNode implementation to explicitly handle visibility fields. Default methods returning null would silently pass contract tests and hide implementations that don't properly store/retrieve visibility data. The migration is mechanical — add two fields/methods to each implementation (InMemoryMindMapStore.StoredNode, SqliteMindMapStore, any downstream).
**Sources:** MindMapNode.java (14 existing methods), InMemoryMindMapStore.StoredNode, SqliteMindMapStore
**Exploration:** quick (surfaced by review)
**Depends on:** D2
**Status:** captured

## D8: PrincipalVisibility accepts String parameters (PrincipalId semantics, zero-deps constraint)

**Choice:** PrincipalVisibility.isVisible() accepts `String callerPrincipalId, String ownerPrincipalId, Set<String> sharedWith`. Callers with PrincipalId pass `.value()`. Callers with ActorId/ParticipantId unwrap to `.principalId().value()`.
**Alternatives:**
- Accept PrincipalId — requires cognitive-api to depend on platform-api, violating zero-deps constraint
- Accept Identity (sealed: PrincipalId | ActorId | ParticipantId) — more permissive than domain requires, same dependency issue
**Rationale:** cognitive-api is the zero-deps cross-cutting shared layer. Adding platform-api would break its architectural constraint for no semantic benefit — the predicate performs string equality and set membership checks that don't require PrincipalId's type structure. All three Identity subtypes produce the same `value()` string (ActorId and ParticipantId delegate to their underlying PrincipalId), so the String signature is unambiguous at runtime. Visibility is semantically about principal ownership — this is encoded by convention (callers pass PrincipalId values), not by parameter type.
**Sources:** MemoryVisibility.java, cognitive-api/pom.xml, Identity.java (sealed: PrincipalId, ActorId, ParticipantId)
**Exploration:** quick (surfaced by review)
**Depends on:** D3
**Status:** captured
