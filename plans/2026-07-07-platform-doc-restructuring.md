# Platform Documentation Restructuring — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task uses verification-driven
> documentation (read current state → write → verify → commit). Steps
> use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #357 — Epic: Platform documentation restructuring
**Issue group:** #357, #13

**Goal:** Decompose PLATFORM.md into LLM-optimized topic chunks, create audience guides with cross-app learning, update all deep-dives and CLAUDE.md files across ~26 repos.

**Architecture:** Monolithic PLATFORM.md → topic-scoped chunks in `docs/platform/`, navigated via `docs/INDEX.md` and two audience guides (`docs/guides/building-apps.md`, `docs/guides/building-platform.md`). Deep-dives stay at full SPI depth. CLAUDE.md files across all repos updated to point to new structure.

**Tech Stack:** Markdown documentation, git multi-repo operations

## Global Constraints

- All paths are relative to `/Users/mdproctor/claude/casehub/parent/` unless stated otherwise
- Commits reference #357: `Refs casehubio/parent#357`
- Every new chunk starts with a scope block header (see spec §Chunk Header Convention)
- Protocol integration uses link-only — one-line scope statement + garden link, no summaries
- building-apps.md must stay under 400 lines
- Topic chunks target 200-500 lines
- Use `git -C /path/to/repo` for all git operations — never `cd`
- Cross-repo CLAUDE.md edits commit to each repo separately
- Read current code in each repo before updating its deep-dive — don't trust the gap list alone

## Dependency Graph

```
Task 1 (scaffold)
  ├─→ Task 2 (extract operational chunks)
  ├─→ Task 3 (extract infrastructure chunks)
  ├─→ Task 4 (new content: notifications, routing, UI arch, CBR)
  └─→ Task 5 (new deep-dive: pages)
        ↓
      Task 6 (INDEX.md) ← depends on Tasks 2-5
        ↓
      Task 7 (building-apps.md) ← depends on Task 6 + deep-dive updates
      Task 8 (building-platform.md) ← depends on Task 6
        ↓
      Task 9  (deep-dive updates: foundation batch)
      Task 10 (deep-dive updates: app + integration batch)
      Task 11 (deep-dive verification)
        ↓
      Task 12 (cross-cutting: names, remapping, retirements)
        ↓
      Task 13 (CLAUDE.md migration: foundation + shared repos)
      Task 14 (CLAUDE.md migration: app + integration repos)
```

Tasks 2-5 can run in parallel after Task 1.
Tasks 7-8 can run in parallel.
Tasks 9-11 can run in parallel.
Tasks 13-14 can run in parallel.

---

### Task 1: Scaffold and PLATFORM.md Redirect

**Files:**
- Create: `docs/platform/` directory
- Create: `docs/guides/` directory
- Modify: `docs/PLATFORM.md` → redirect

**Interfaces:**
- Consumes: nothing
- Produces: directory structure + PLATFORM.md redirect that all subsequent tasks depend on

- [ ] **Step 1: Create directory structure**

```bash
mkdir -p /Users/mdproctor/claude/casehub/parent/docs/platform
mkdir -p /Users/mdproctor/claude/casehub/parent/docs/guides
```

- [ ] **Step 2: Back up PLATFORM.md content**

Read `docs/PLATFORM.md` in full. This is the source material for Tasks 2-3. Save a copy:

```bash
cp /Users/mdproctor/claude/casehub/parent/docs/PLATFORM.md /Users/mdproctor/claude/casehub/parent/docs/PLATFORM-backup.md
```

- [ ] **Step 3: Write PLATFORM.md redirect**

Replace `docs/PLATFORM.md` with:

```markdown
# CaseHub Platform Documentation

This document has been decomposed into topic-scoped chunks for LLM efficiency.

**Start here:** [INDEX.md](INDEX.md) — discovery index organized by concern

**Audience guides:**
- [Building Apps](guides/building-apps.md) — for application developers
- [Building Platform](guides/building-platform.md) — for platform contributors

For the full topic chunk listing, see INDEX.md.
```

- [ ] **Step 4: Verify the redirect**

Confirm the redirect file exists and links are syntactically correct (targets don't exist yet — that's expected).

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/platform/.gitkeep docs/guides/.gitkeep docs/PLATFORM.md docs/PLATFORM-backup.md
git commit -m "chore(#357): scaffold platform doc restructuring — redirect PLATFORM.md, create dirs

Refs casehubio/parent#357"
```

---

### Task 2: Extract Operational Chunks from PLATFORM.md

**Files:**
- Create: `docs/platform/coherence-protocol.md`
- Create: `docs/platform/capability-ownership.md`
- Create: `docs/platform/dependency-map.md`
- Create: `docs/platform/boundary-rules.md`
- Create: `docs/platform/overlap-risks.md`
- Read: `docs/PLATFORM-backup.md` (source material)

**Interfaces:**
- Consumes: `PLATFORM-backup.md` (from Task 1)
- Produces: 5 topic chunks linked by INDEX.md (Task 6)

- [ ] **Step 1: Read PLATFORM-backup.md to identify source sections**

Read the full backup file. Identify these sections:
- Lines 13-98: Platform Coherence Protocol (Steps 1-6)
- Lines 361-474: Capability Ownership table
- Lines 219-354: Cross-Repo Dependency Map (Repository Map, Build Order, Cross-Repo Dependencies)
- Key Boundary Rules section (all "do not" rules)
- Known Overlap Risks + Known Placement Violations sections

- [ ] **Step 2: Write coherence-protocol.md**

Extract Steps 1-6. Add scope header:
```markdown
# Platform Coherence Protocol

> **Scope:** The 6-step pre-implementation protocol — run before any platform change
> **Audience:** All (platform + app builders)
> **Key repos:** all casehubio repos
```

For Step 4 (implementation conventions): replace inline pattern summaries with one-line scope statements + links to garden protocols. This is consistent with the link-only principle.

Estimated: ~100 lines (Steps 1-3 + 5-6 are procedural; Step 4 is curated links).

- [ ] **Step 3: Write capability-ownership.md**

Extract the Capability Ownership table. Add scope header:
```markdown
# Capability Ownership

> **Scope:** "Where does X live?" — authoritative lookup for capability → repo mapping
> **Audience:** All
```

Estimated: ~120 lines.

- [ ] **Step 4: Write dependency-map.md**

Extract Repository Map, Build Order, and Cross-Repo Dependencies. Add scope header:
```markdown
# Cross-Repo Dependency Map

> **Scope:** Repo topology, build order, and dependency impact analysis
> **Audience:** Platform builders (app builders rarely need this)
> **Key repos:** all foundation repos
```

Estimated: ~140 lines.

- [ ] **Step 5: Write boundary-rules.md**

Consolidate ALL "do not" rules from the Key Boundary Rules section. Add scope header:
```markdown
# Boundary Rules

> **Scope:** All cross-repo "do not" rules — what must NOT be placed where
> **Audience:** All
```

Estimated: ~250 lines.

- [ ] **Step 6: Write overlap-risks.md**

Consolidate Known Overlap Risks + Known Placement Violations. Add scope header:
```markdown
# Known Overlap Risks

> **Scope:** Semantic collisions between repos and capabilities in wrong modules
> **Audience:** Platform builders primarily; app builders for awareness
```

Estimated: ~150 lines.

- [ ] **Step 7: Verify all 5 chunks**

For each chunk:
1. Confirm the scope header follows the convention
2. Confirm content matches the PLATFORM-backup.md source (no accidental omission)
3. Confirm no protocol summaries — only link-only references

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/platform/coherence-protocol.md docs/platform/capability-ownership.md docs/platform/dependency-map.md docs/platform/boundary-rules.md docs/platform/overlap-risks.md
git commit -m "docs(#357): extract operational chunks from PLATFORM.md

coherence-protocol, capability-ownership, dependency-map,
boundary-rules, overlap-risks — all extracted with scope headers.

Refs casehubio/parent#357"
```

---

### Task 3: Extract Infrastructure Chunks from PLATFORM.md

**Files:**
- Create: `docs/platform/overview.md`
- Create: `docs/platform/persistence.md`
- Create: `docs/platform/auth.md`
- Create: `docs/platform/observability.md`
- Create: `docs/platform/agent-identity.md`
- Create: `docs/platform/agent-mesh.md`
- Create: `docs/platform/privacy.md`
- Create: `docs/platform/protocols.md`
- Read: `docs/PLATFORM-backup.md` (source material)
- Read: garden protocols INDEX.md (for protocol links)

**Interfaces:**
- Consumes: `PLATFORM-backup.md` (from Task 1)
- Produces: 8 topic chunks linked by INDEX.md (Task 6)

- [ ] **Step 1: Read PLATFORM-backup.md to identify source sections**

Read the full backup. Identify:
- Tier structure, build order → overview.md
- Upstream Consistency / Serverless Workflow 1.0 (lines 107-117) → overview.md
- Persistence section → persistence.md
- Authentication + Outbound Authentication + Role Name Convention → auth.md
- Observability section → observability.md
- Agent Identity section → agent-identity.md
- Agent Communication Mesh section → agent-mesh.md
- Privacy (GDPR) section → privacy.md
- Implementation Protocols table → protocols.md

- [ ] **Step 2: Read garden protocols INDEX.md**

```bash
cat /Users/mdproctor/.hortora/garden/docs/protocols/INDEX.md
```

Collect all protocol names, scopes, and file paths for linking.

- [ ] **Step 3: Write overview.md**

Tier structure, repo map, build order. Include upstream consistency (Serverless Workflow 1.0 / quarkus-flow constraint). Scope header:
```markdown
# Platform Overview

> **Scope:** Tier architecture, repository map, build order, upstream framework alignment
> **Audience:** All
```

Estimated: ~200 lines.

- [ ] **Step 4: Write persistence.md**

Flyway conventions, datasource naming, migration paths. Link to flyway protocols. Scope header:
```markdown
# Persistence

> **Scope:** Flyway migration conventions, datasource naming, migration path scoping
> **Audience:** All (platform + app builders)
> **Key repos:** casehub-ledger, casehub-work, casehub-qhorus, casehub-engine
> **Protocols:** flyway-repo-scoped-migration-path, flyway-migration-rules,
>   flyway-extension-migration-registration
```

Estimated: ~200 lines.

- [ ] **Step 5: Write auth.md**

Gateway topology, roles, outbound credentials (3 tiers), secrets management. Scope header:
```markdown
# Authentication & Authorization

> **Scope:** Gateway topology, role conventions, outbound credential tiers, secrets management
> **Audience:** All
> **Key repos:** claudony (gateway), casehub-platform (OIDC), casehub-qhorus (channel ACL)
> **Protocols:** auth-retrofit-readiness, per-binding-credential-reference
```

Estimated: ~250 lines.

- [ ] **Step 6: Write observability.md**

OTel trace → ledger, agent interaction audit, WorkItem audit, case decision logging. Scope header:
```markdown
# Observability

> **Scope:** OTel tracing, audit trails, agent interaction recording, case decision logs
> **Audience:** All
> **Key repos:** casehub-ledger, casehub-work, casehub-engine, casehub-qhorus
> **Protocols:** dual-trail-audit-pattern
```

Estimated: ~150 lines.

- [ ] **Step 7: Write agent-identity.md**

DID format, SCIM2, eidos descriptors, agent versioning. Scope header:
```markdown
# Agent Identity

> **Scope:** DID format, SCIM2 resolution, agent descriptor structure, versioning
> **Audience:** All
> **Key repos:** casehub-ledger (DID), casehub-eidos (descriptors), casehub-platform (identity)
> **Protocols:** scim2-agent-identity
```

Estimated: ~150 lines.

- [ ] **Step 8: Write agent-mesh.md**

3-channel layout, speech acts, mesh participation, normative accountability framework. Scope header:
```markdown
# Agent Communication Mesh

> **Scope:** Normative 3-channel layout (work/observe/oversight), mesh participation strategies
> **Audience:** All
> **Key repos:** casehub-qhorus (transport), casehub-engine (orchestration), casehub-engine-api (layout SPIs)
```

Estimated: ~200 lines.

- [ ] **Step 9: Write privacy.md**

GDPR erasure, PII sanitisation, decision records. Scope header:
```markdown
# Privacy (GDPR)

> **Scope:** Art.17 erasure, Art.22 decision records, PII sanitisation
> **Audience:** All (critical for regulated-domain app builders)
> **Key repos:** casehub-ledger
```

Estimated: ~100 lines.

- [ ] **Step 10: Write protocols.md**

Protocol summary with audience mapping. Link to garden INDEX.md. Include two sections: "For App Builders" and "For Platform Builders." Scope header:
```markdown
# Implementation Protocols

> **Scope:** Cross-cutting conventions — audience-mapped summary with garden links
> **Audience:** All
> **Full index:** garden protocols INDEX.md
```

Estimated: ~150 lines.

- [ ] **Step 11: Verify all 8 chunks**

For each chunk: confirm scope header, confirm content extracted correctly, confirm protocol references are link-only (no summaries), confirm no PLATFORM.md content was accidentally omitted.

- [ ] **Step 12: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/platform/overview.md docs/platform/persistence.md docs/platform/auth.md docs/platform/observability.md docs/platform/agent-identity.md docs/platform/agent-mesh.md docs/platform/privacy.md docs/platform/protocols.md
git commit -m "docs(#357): extract infrastructure chunks from PLATFORM.md

overview, persistence, auth, observability, agent-identity,
agent-mesh, privacy, protocols — all with scope headers.

Refs casehubio/parent#357"
```

---

### Task 4: Write New Topic Chunks (Undocumented Capabilities)

**Files:**
- Create: `docs/platform/notifications.md`
- Create: `docs/platform/routing.md`
- Create: `docs/platform/ui-architecture.md`
- Create: `docs/platform/cbr.md`
- Read: `docs/CBR-CAPABILITY.md` (to absorb)
- Read: current code in platform, blocks, engine, pages, blocks-ui repos

**Interfaces:**
- Consumes: current code across repos (primary source — these capabilities are undocumented)
- Produces: 4 topic chunks linked by INDEX.md (Task 6)

These chunks require reading current code — they document capabilities that don't exist in PLATFORM.md. This is the most creative task.

- [ ] **Step 1: Research notification system in platform repo**

Read the platform repo to understand the notification/subscription system (platform#142-149):

```bash
git -C /Users/mdproctor/claude/casehub/platform log --oneline -30 --grep="notification\|subscription\|EventTypeRegistry" 
```

Use IntelliJ MCP to find key classes:
- `EventTypeRegistry` SPI
- Subscription engine classes
- Notification delivery pipeline
- Digest batching mechanism

- [ ] **Step 2: Write notifications.md**

```markdown
# Notifications

> **Scope:** Subscription engine, notification delivery pipeline, digest batching
> **Audience:** All (app builders who want to send notifications; platform builders extending the pipeline)
> **Key repos:** casehub-platform (subscription engine), casehub-connectors (delivery channels)
```

Document: EventTypeRegistry SPI, subscription model, delivery pipeline, target resolution, suppression, channel routing, digest batching. ~200 lines.

- [ ] **Step 3: Research routing architecture across repos**

Read routing code across blocks, engine, and neocortex:
- blocks `io.casehub.blocks.routing` and `io.casehub.blocks.routing.agent` packages
- engine `TrustWeightedAgentStrategy`, `SemanticAgentRoutingStrategy`
- ledger `TrustScoreRoutingPublisher`
- The 4-layer ownership model from blocks CLAUDE.md §Trust Routing Architecture

- [ ] **Step 4: Write routing.md**

```markdown
# Routing

> **Scope:** Agent routing strategies — trust-weighted, semantic, LLM-reasoned, CBR-evidence
> **Audience:** All (app builders configure routing; platform builders extend strategies)
> **Key repos:** casehub-ledger (score computation), casehub-blocks (policy config + AI strategies),
>   casehub-engine (classical strategy execution)
```

Document the 4-layer ownership model, strategy types, RoutingFeatureExtractor SPI, RoutingPromptSection SPI, RoutingOutcomeRecorder. ~250 lines.

- [ ] **Step 5: Research UI architecture across pages, blocks-ui, and app repos**

Read current state of:
- casehub-pages: package structure, ConfigurablePanel, DataReceiver, pages-data, pages-component
- casehub-blocks-ui: all 14 components, maturity levels, pages integration
- App UIs: how AML, DevTown, Drafthouse compose their UIs

- [ ] **Step 6: Write ui-architecture.md**

```markdown
# UI Architecture

> **Scope:** pages → blocks-ui → app UI layering, composition patterns, component catalogue
> **Audience:** App builders primarily; platform builders extending pages/blocks-ui
> **Key repos:** casehub-pages (infrastructure), casehub-blocks-ui (domain components)
> **Protocols:** custom-event-shadow-dom, lit-immutable-collections
```

Document: 3-layer model (pages → blocks-ui → app), composition pattern (Quinoa + pages runtime + hostPanel), ConfigurablePanel interface, DataReceiver contract, blocks-ui component catalogue with maturity levels, how apps compose UIs. ~300 lines.

- [ ] **Step 7: Write cbr.md (absorbing CBR-CAPABILITY.md)**

Read `docs/CBR-CAPABILITY.md`. Read current CBR code across engine, neocortex, clinical, blocks. Produce a combined chunk:

```markdown
# Case-Based Reasoning (CBR)

> **Scope:** The 4 CBR steps (Retrieve, Reuse, Revise, Retain) and their platform homes
> **Audience:** All
> **Key repos:** casehub-neocortex (retrieval), casehub-engine (reuse), casehub-ledger (retain),
>   casehub-blocks (feature extraction)
```

~200 lines. Must be strictly better than CBR-CAPABILITY.md — incorporate all recent CBR integration work (CbrRetrievalService, RoutingFeatureExtractor, CbrRoutingOutcomeRecorder, HybridCaseRetriever).

- [ ] **Step 8: Verify all 4 chunks**

For each chunk: confirm scope header, confirm content reflects current code (not stale), confirm protocol references are link-only.

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/platform/notifications.md docs/platform/routing.md docs/platform/ui-architecture.md docs/platform/cbr.md
git commit -m "docs(#357): write new topic chunks — notifications, routing, UI architecture, CBR

New content for capabilities not previously documented in PLATFORM.md.
Each chunk reflects current code state across relevant repos.

Refs casehubio/parent#357"
```

---

### Task 5: Write casehub-pages Deep-Dive

**Files:**
- Create: `docs/repos/casehub-pages.md`
- Read: casehub-pages repo (package structure, key abstractions, recent changes)

**Interfaces:**
- Consumes: casehub-pages repo code
- Produces: deep-dive linked by INDEX.md (Task 6) and ui-architecture.md (Task 4)

- [ ] **Step 1: Read casehub-pages repo structure**

```bash
git -C /Users/mdproctor/claude/casehub/pages log --oneline -30
```

Read `package.json` for workspace structure. Read key source files for:
- Core packages (pages-ui, pages-data, pages-viz, pages-component, pages-runtime)
- Iframe API (pages-iframe-api, pages-iframe-dev)
- Component packages (llm-prompter, svg-heatmap, terminal)
- Key interfaces: ConfigurablePanel, DataReceiver, VizTarget

- [ ] **Step 2: Write casehub-pages.md**

Follow the established deep-dive structure (header, purpose, module structure, key abstractions, dependencies, depended on by). Include:
- Package structure table (10 core packages + 3 iframe components)
- Key abstractions: ConfigurablePanel, DataReceiver, DataSet model, pages-event system
- Composition pattern: loadSite() → navigation → data pipeline → panel hosting
- Quinoa integration pattern
- Recent evolution: melviz → casehub-pages rename, primitives removal

~300 lines.

- [ ] **Step 3: Verify deep-dive**

Cross-check against actual repo structure. Confirm all package names match current code. Confirm key interfaces exist and signatures are current.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/casehub-pages.md
git commit -m "docs(#357): add casehub-pages deep-dive

New deep-dive for the web component framework. Covers package structure,
ConfigurablePanel/DataReceiver interfaces, data pipeline, Quinoa integration.

Refs casehubio/parent#357"
```

---

### Task 6: Write INDEX.md

**Files:**
- Create: `docs/INDEX.md`
- Read: all files created in Tasks 2-5 (to link correctly)

**Interfaces:**
- Consumes: all topic chunks (Tasks 2-5) and existing deep-dives
- Produces: discovery index used by all CLAUDE.md files (Tasks 13-14) and guides (Tasks 7-8)

- [ ] **Step 1: Inventory all linkable targets**

List every file in `docs/platform/`, `docs/repos/`, and standalone docs. Each becomes an index entry.

- [ ] **Step 2: Write INDEX.md**

Follow the structure from the spec §INDEX.md Content Structure. Organize by concern, not by repo. Each entry: capability name → one-line description → link. Include protocol names inline with relevant topics.

Add the header:
```markdown
# CaseHub Platform Index

> Load this first. Find your topic, follow the link.
> For audience-specific guidance: [Building Apps](guides/building-apps.md)
> | [Building Platform](guides/building-platform.md)
```

Target: ~150 lines.

- [ ] **Step 3: Verify all links resolve**

For every `→ [path]` in INDEX.md, confirm the target file exists on disk.

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/INDEX.md
git commit -m "docs(#357): create discovery index — INDEX.md

Universal LLM entry point organized by concern. Links to all topic chunks,
deep-dives, and standalone docs. ~150 lines.

Refs casehubio/parent#357"
```

---

### Task 7: Write building-apps.md

**Files:**
- Create: `docs/guides/building-apps.md`
- Read: `docs/INDEX.md`, all deep-dives in `docs/repos/`, `docs/platform/ui-architecture.md`, `docs/platform/routing.md`
- Read: blocks CLAUDE.md §Blocks Scope Criteria, blocks-ui CLAUDE.md §Design Philosophy

**Interfaces:**
- Consumes: INDEX.md (Task 6), topic chunks (Tasks 2-5), deep-dives
- Produces: app builder guide linked by INDEX.md and app repo CLAUDE.md files (Task 14)

- [ ] **Step 1: Read all app deep-dives for cross-referencing**

Read deep-dives for: AML, clinical, devtown, life, drafthouse, quarkmind, soc, fsitrading. Note layer progression, completed capabilities, and UI status for each.

- [ ] **Step 2: Build the capability matrix**

Create the grid: shared building blocks (blocks + blocks-ui) first, then per-app implementation status with layer numbers. Include the "Starting a new app?" decision tree.

- [ ] **Step 3: Write the pattern catalogue**

Compact format: one line for shared pattern + link to topic chunk, brief per-app references linking to deep-dive sections. Cover: case types, trust routing, oversight gates, web UI composition, GDPR erasure, notifications, CBR.

- [ ] **Step 4: Write placement criteria**

Table covering blocks (Java), blocks-ui (TypeScript), platform, engine, app repos. Reference blocks scope criteria. Include worked examples from consolidation epic #28.

- [ ] **Step 5: Write protocols section and layer progression**

Protocols for app builders (grouped by concern). Layer progression table with cross-references to each app's ARC42STORIES.MD.

- [ ] **Step 6: Assemble building-apps.md**

Combine all sections. Verify total is under 400 lines. If over, push per-app detail into deep-dive cross-references (not inline content).

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/guides/building-apps.md
git commit -m "docs(#357): create app builder guide — building-apps.md

Capability matrix, pattern catalogue, placement criteria, cross-app
learning references. Anti-blind-spot document for app developers.

Refs casehubio/parent#357"
```

---

### Task 8: Write building-platform.md

**Files:**
- Create: `docs/guides/building-platform.md`
- Read: `docs/INDEX.md`, `docs/platform/boundary-rules.md`, `docs/platform/overlap-risks.md`
- Read: garden protocols INDEX.md

**Interfaces:**
- Consumes: INDEX.md (Task 6), topic chunks
- Produces: platform builder guide linked by INDEX.md and platform repo CLAUDE.md files (Task 13)

- [ ] **Step 1: Write building-platform.md**

Follow the spec §`guides/building-platform.md` structure:
1. Architecture links
2. Boundary rules links
3. SPI design conventions (protocol links)
4. "Adding a new capability" checklist
5. Cross-cutting concern links
6. All protocols (full list)
7. Application impact reminder

Target: ~200 lines. Mostly navigation + the "adding a new capability" checklist.

- [ ] **Step 2: Verify all links resolve**

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/guides/building-platform.md
git commit -m "docs(#357): create platform builder guide — building-platform.md

Navigation guide for platform contributors: boundary rules, SPI conventions,
capability addition checklist, protocol index.

Refs casehubio/parent#357"
```

---

### Task 9: Update Foundation Deep-Dives (Batch 1)

**Files:**
- Modify: `docs/repos/casehub-platform.md`
- Modify: `docs/repos/casehub-engine.md`
- Modify: `docs/repos/casehub-ledger.md`
- Modify: `docs/repos/casehub-work.md`
- Modify: `docs/repos/casehub-qhorus.md`
- Modify: `docs/repos/casehub-eidos.md`
- Modify: `docs/repos/casehub-connectors.md`
- Modify: `docs/repos/casehub-neocortex.md`
- Modify: `docs/repos/casehub-workers.md`

**Interfaces:**
- Consumes: current code in each repo
- Produces: updated deep-dives accurate to current code state

For EACH deep-dive in this batch:

- [ ] **Step 1: Read current deep-dive**

- [ ] **Step 2: Read current repo code**

Use `git log --oneline -30` on the repo + IntelliJ MCP (`ide_find_class`, `ide_file_structure`) to identify what's changed since the deep-dive was last updated.

- [ ] **Step 3: Identify gaps**

Compare the deep-dive against current code. Specific gaps per the spec:
- **platform:** notification/subscription system, EventTypeRegistry, DataSource SPI
- **engine:** CBR bridge, types/labels, RoutingOutcomeRecorder, CaseDefinition classification
- **ledger:** ledger-rest extraction, cloud KMS signers (AWS/GCP/Azure/Vault)
- **work:** types/labels, tenancy-aware query, work-rest module extraction
- **qhorus:** verify slack channel backend, postgres broadcaster, inbound bridge
- **eidos:** behavioral contracts, semantic capability matching, capability descriptions
- **connectors:** ChatPlatform SPI, RichCard, chat demo, Discord translation
- **neocortex:** fix stale names (CorpusStore→EmbeddingIngestor etc.), add Matryoshka, quantization, CBR reconciliation
- **workers:** K8s worker dispatch, restart recovery, bindingName correlation

- [ ] **Step 4: Update the deep-dive**

Add missing modules, update stale type names, add new SPIs and abstractions. Maintain the established deep-dive structure (header, purpose, module structure, key abstractions, dependencies, consumers).

- [ ] **Step 5: Verify updated content against code**

For each new class/SPI added to the deep-dive: confirm it exists in current code with the documented name and signature.

- [ ] **Step 6: Commit each deep-dive separately**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/repos/<name>.md
git commit -m "docs(#357): update <name> deep-dive — <brief summary of additions>

Refs casehubio/parent#357"
```

---

### Task 10: Update App & Integration Deep-Dives (Batch 2)

**Files:**
- Modify: `docs/repos/casehub-blocks.md`
- Modify: `docs/repos/casehub-blocks-ui.md`
- Modify: `docs/repos/casehub-clinical.md`
- Modify: `docs/repos/casehub-aml.md`
- Modify: `docs/repos/casehub-devtown.md`
- Modify: `docs/repos/casehub-drafthouse.md`
- Modify: `docs/repos/claudony.md`
- Modify: `docs/repos/casehub-ras.md`
- Modify: `docs/repos/casehub-desiredstate.md`
- Modify: `docs/repos/casehub-ops.md`

**Interfaces:**
- Consumes: current code in each repo
- Produces: updated deep-dives accurate to current code state

Same process as Task 9. Specific gaps per the spec:
- **blocks:** replace "scaffold" status with full module structure (6 packages), scope criteria, consumers table, consolidation epic #28
- **blocks-ui:** add notification-inbox, data-table, work-item-workbench; add maturity levels; add design philosophy
- **clinical:** CBR integration, ClinicalRoutingFeatureExtractor, REST precedent endpoints
- **aml:** web UI views (operations, accountability, investigations, datasets)
- **devtown:** governance workbench UI, WebSocket bridge, GovernanceQueryService
- **drafthouse:** replay adapter, WebSocket reconnection, section highlighting, Quinoa
- **claudony:** pages/Quinoa adoption, multi-tenancy, ProvisionerConfigRegistry
- **ras:** verify against current code (Ganglion SPIs, DroolsGanglion, case triggers)
- **desiredstate:** verify against current code (LifecycleManager, PendingApproval, phase API)
- **ops:** verify against current code (EvidenceCollector, ops console, topology manager)

Follow the same read → gap → update → verify → commit cycle as Task 9.

---

### Task 11: Verify Remaining Deep-Dives

**Files:**
- Verify: `docs/repos/casehub-worker.md`
- Verify: `docs/repos/casehub-iot.md`
- Verify: `docs/repos/casehub-identity.md`
- Verify: `docs/repos/casehub-life.md`
- Verify: `docs/repos/casehub-openclaw.md`
- Verify: `docs/repos/casehub-fsitrading.md`
- Verify: `docs/repos/casehub-soc.md`
- Verify: `docs/repos/quarkmind.md`

**Interfaces:**
- Consumes: current code in each repo
- Produces: verified deep-dives (updated if stale, confirmed if current)

For EACH deep-dive:

- [ ] **Step 1: Read the deep-dive and compare against current repo code**

Use `git log --oneline -20` on the repo to see recent changes. Quick-scan for:
- New modules not in the deep-dive
- Renamed types
- New SPIs or major abstractions
- Changed module structure

- [ ] **Step 2: If current → no action. If stale → update and commit.**

---

### Task 12: Cross-Cutting Updates (Names, Remapping, Retirements)

**Files:**
- Modify: multiple deep-dives (name fixes)
- Modify: `docs/config-architecture.md` (remapping)
- Modify: `docs/arc42stories-casehub-profile.md` (remapping)
- Modify: `docs/ARCHITECTURE.md` (header fix)
- Modify: `docs/APPLICATIONS.md` (UI status, cross-references)
- Modify: `docs/prompt-snippets.md` (new structure reference)
- Delete: `docs/CBR-CAPABILITY.md` (retired — absorbed into platform/cbr.md)
- Delete: `docs/PLATFORM-backup.md` (cleanup)

**Interfaces:**
- Consumes: all files from Tasks 2-11
- Produces: fully consistent documentation with no stale references

- [ ] **Step 1: Fix cross-cutting stale names**

Search all deep-dives for the 8 name patterns from the spec §Cross-Cutting Name Fixes. Use `grep -r` to find occurrences, then edit each file.

- [ ] **Step 2: Remap config-architecture.md**

Follow the spec §config-architecture.md Remapping table. Every PLATFORM.md topic ownership entry → new chunk location.

- [ ] **Step 3: Remap arc42stories-casehub-profile.md**

Follow the spec §arc42stories-casehub-profile.md Remapping table. 6 section-specific references → new chunk locations.

- [ ] **Step 4: Update ARCHITECTURE.md header**

Replace "Supplement to PLATFORM.md" with standalone description referencing the new structure.

- [ ] **Step 5: Update APPLICATIONS.md**

Add UI status column, blocks-ui component usage, cross-references to building-apps.md pattern catalogue.

- [ ] **Step 6: Update prompt-snippets.md**

Change "consult PLATFORM.md" → "consult INDEX.md + relevant chunks."

- [ ] **Step 7: Retire CBR-CAPABILITY.md**

Delete the file (content absorbed into `platform/cbr.md`).

- [ ] **Step 8: Delete PLATFORM-backup.md**

```bash
git -C /Users/mdproctor/claude/casehub/parent rm docs/PLATFORM-backup.md docs/CBR-CAPABILITY.md
```

- [ ] **Step 9: Verify no remaining stale PLATFORM.md references**

```bash
grep -r "PLATFORM\.md" /Users/mdproctor/claude/casehub/parent/docs/ --include="*.md" | grep -v "PLATFORM.md:" | grep -v "PLATFORM-backup"
```

Any remaining references should point to the redirect (expected) or be section-specific (unexpected — fix them).

- [ ] **Step 10: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add -A docs/
git commit -m "docs(#357): cross-cutting updates — name fixes, remapping, retirements

Fix 8 stale name patterns across deep-dives. Remap config-architecture.md
and arc42stories references. Update ARCHITECTURE.md header. Retire
CBR-CAPABILITY.md (absorbed into platform/cbr.md).

Refs casehubio/parent#357"
```

---

### Task 13: CLAUDE.md Migration — Foundation + Shared Repos

**Files:**
- Modify: CLAUDE.md in ~19 repos: platform, engine, ledger, work, qhorus, eidos, connectors, neocortex, workers, worker, iot, ras, desiredstate, ops, openclaw, flow, blocks, blocks-ui, pages

**Interfaces:**
- Consumes: INDEX.md (Task 6), building-platform.md (Task 8), building-apps.md (Task 7)
- Produces: updated CLAUDE.md files pointing to new doc structure

For EACH repo:

- [ ] **Step 1: Read current CLAUDE.md**

Find the section that references PLATFORM.md (usually under "Platform Context" or "Platform doc").

- [ ] **Step 2: Update platform doc references**

Replace PLATFORM.md references with:

For **platform/foundation repos:**
```markdown
## Platform Docs
- [Platform Index](https://raw.githubusercontent.com/casehubio/parent/main/docs/INDEX.md) — discovery index (start here)
- [Building Platform](https://raw.githubusercontent.com/casehubio/parent/main/docs/guides/building-platform.md) — platform contributor guide
- [This repo's deep-dive](https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-<name>.md)
```

For **shared pattern repos** (blocks, blocks-ui, pages): same as above plus:
```markdown
- [UI Architecture](https://raw.githubusercontent.com/casehubio/parent/main/docs/platform/ui-architecture.md)
```

- [ ] **Step 3: Commit to each repo separately**

```bash
git -C /Users/mdproctor/claude/casehub/<repo> add CLAUDE.md
git -C /Users/mdproctor/claude/casehub/<repo> commit -m "docs(#357): update CLAUDE.md — point to new platform doc structure

Replace PLATFORM.md reference with INDEX.md + building-platform.md.

Refs casehubio/parent#357"
```

---

### Task 14: CLAUDE.md Migration — App + Integration Repos

**Files:**
- Modify: CLAUDE.md in ~9 repos: aml, clinical, devtown, life, drafthouse, quarkmind, soc, fsitrading, claudony

**Interfaces:**
- Consumes: INDEX.md (Task 6), building-apps.md (Task 7)
- Produces: updated CLAUDE.md files pointing to new doc structure

Same process as Task 13, but with app-builder references:

For **app repos:**
```markdown
## Platform Docs
- [Platform Index](https://raw.githubusercontent.com/casehubio/parent/main/docs/INDEX.md) — discovery index (start here)
- [Building Apps](https://raw.githubusercontent.com/casehubio/parent/main/docs/guides/building-apps.md) — app developer guide with cross-app patterns
- [This repo's deep-dive](https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/casehub-<name>.md)
```

For **claudony** (integration): include both guides:
```markdown
## Platform Docs
- [Platform Index](https://raw.githubusercontent.com/casehubio/parent/main/docs/INDEX.md) — discovery index (start here)
- [Building Apps](https://raw.githubusercontent.com/casehubio/parent/main/docs/guides/building-apps.md) — app developer guide
- [Building Platform](https://raw.githubusercontent.com/casehubio/parent/main/docs/guides/building-platform.md) — platform contributor guide
- [This repo's deep-dive](https://raw.githubusercontent.com/casehubio/parent/main/docs/repos/claudony.md)
```

Commit each repo separately as in Task 13.

---

## Execution Notes

**Parallelization:** Tasks 2-5 are fully independent after Task 1. Tasks 7-8 are independent. Tasks 9-11 are independent. Tasks 13-14 are independent. Use subagent-driven-development with parallel dispatch for these groups.

**Largest tasks by effort:** Tasks 9 and 10 (deep-dive updates) are the heaviest — each requires reading code across multiple repos. Consider splitting each into sub-tasks (one per repo) if subagent context limits are hit.

**Task 4 is the most creative:** It writes new content for undocumented capabilities. The subagent needs access to all repo directories to read current code.

**Cross-repo commits (Tasks 13-14):** Each repo gets its own commit. Do not batch commits across repos. Push all repos after all commits succeed.
