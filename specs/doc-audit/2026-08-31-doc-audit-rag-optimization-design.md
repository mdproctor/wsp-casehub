# Documentation Audit, RAG Optimization & Freshness Gate — Design Spec

**Date:** 2026-08-31
**Branch:** doc-audit (to be created)
**Covers:** Platform-wide documentation freshness, RAG retrieval quality, work-end enforcement
**Status:** Draft

---

## 1. Problem Statement

CaseHub has 28 repos with 1,700+ commits of drift since consumer/contributor guides were last updated (Aug 3-10, 2026). The documentation structure — INDEX.md → consumer/contributor indexes → per-repo guides + arc42stories — is comprehensive but stale. An LLM starting a session cannot trust that guides accurately describe current capabilities, SPIs, or module structures.

Three interconnected problems:
1. **Staleness** — guides don't reflect current code
2. **RAG inefficiency** — monolithic 200-550 line guides load unnecessary content and chunk poorly for retrieval
3. **No enforcement** — nothing prevents docs from falling behind as code evolves

---

## 2. Deliverables

| # | Deliverable | Lifecycle |
|---|-------------|-----------|
| D-A | Initial audit — identify all stale sections across 28 repos | One-time (audit slot) |
| D-B | Fix all identified staleness using git history, issues, and blogs as sources | One-time (audit slot, per-repo sessions) |
| D-C | RAG optimization — decompose guides, add structural anchors, create capability index | Incremental (during D-B, largest guides first) |
| D-D | Work-end freshness gate — adversarial enforcement preventing future drift | Permanent (skill modification) |

---

## 3. Architecture

### 3.1 Structural Anchors (D4) — the foundation

Documentation sections declare the code elements they describe via **structural anchors** — class names, SPI interfaces, config keys, file paths, protocol references. These serve dual purpose:

1. **Staleness detection:** When an anchored element changes (detected via git diff), the anchoring section is flagged as candidate-stale
2. **RAG metadata:** Anchors appear in YAML frontmatter, enabling precise filtered retrieval via neocortex's PayloadFilter

Anchor format in documentation YAML frontmatter:
```yaml
---
capability: notifications
audience: consumer
repo: casehub-platform
anchors:
  classes:
    - io.casehub.platform.notification.NotificationBridge
    - io.casehub.platform.notification.SubscriptionEngine
  spis:
    - io.casehub.platform.notification.spi.DeliveryChannel
  config-keys:
    - casehub.notification.digest.interval
  protocols:
    - flyway-version-range-allocation
    - notification-delivery-contract
---
```

When any anchored element is renamed, moved, or deleted, the diff-based triage detects the change and flags the section for adversarial verification.

**Override mechanism:** A `verified-current: 2026-08-31` annotation suppresses re-flagging until the next anchor change. Used when an LLM adversarial check confirms the section is accurate despite an anchor change (e.g., internal refactoring that doesn't affect the documented behavior).

### 3.2 Three-Layer Document Structure (D5)

Following the PLATFORM.md decomposition precedent (topic files + thin index), guides are incrementally decomposed:

**Layer 1 — Capability Index** (`docs/capabilities.md`)
A single cross-repo file mapping capabilities to locations. The RAG entry point for ad-hoc retrieval:

```markdown
| Capability | Repo | Audience | Chunk |
|------------|------|----------|-------|
| Send notifications | platform | consumer | repos/casehub-platform/capabilities/notifications.md |
| Case lifecycle | engine | consumer | repos/casehub-engine/capabilities/case-lifecycle.md |
| Trust routing | ledger | consumer | repos/casehub-ledger/capabilities/trust-routing.md |
```

**Layer 2 — Per-Capability Chunks** (`docs/repos/<repo>/capabilities/<topic>.md`)
Self-contained files with YAML frontmatter. Each chunk is independently loadable for RAG retrieval:

```
docs/repos/casehub-platform/
├── capabilities/
│   ├── notifications.md        (extracted from consumer-guide § Notifications)
│   ├── identity.md             (extracted from consumer-guide § Identity)
│   ├── expressions.md          (extracted from consumer-guide § Expressions)
│   └── ...
├── internals/
│   ├── notification-pipeline.md (extracted from contributor-guide § Notification Dispatch)
│   └── ...
├── consumer-guide.md           (thin index linking to capabilities/)
└── contributor-guide.md        (thin index linking to internals/)
```

**Layer 3 — Guide Indexes** (existing `consumer-guide.md` / `contributor-guide.md`)
Become thin routing documents: a brief repo overview + links to capability chunks. For session loading, an LLM reads the guide index and pulls in specific chunks as needed.

**Decomposition priority:** Largest guides first — devtown (546 lines), qhorus (526 lines), ops (474 lines), ledger (448 lines), platform (418 lines), engine (408 lines). Extract sections exceeding 40 lines into standalone capability chunks (typically 3-5 per guide). Smaller guides (fsitrading at 119 lines, quarkmind at 186 lines) stay monolithic unless retrieval failures justify decomposition.

### 3.3 Staleness Detection — Hybrid Methodology (D4)

Three-phase detection:

**Phase 1 — Structural anchor check (mechanical, fast)**
For each file changed in the branch diff, check whether any documentation section anchors that file's classes, SPIs, or config keys. If yes, flag the section as candidate-stale.

**Phase 2 — Diff-based triage (transition fallback)**
For documentation sections without structural anchors (during the audit transition period), scan all un-anchored sections when any file in the repo changes. This is intentionally coarse — it trades precision for coverage. Sunset condition: once all sections have anchors (D3 exit criteria), diff-based triage is retired.

**Phase 3 — LLM adversarial verification (semantic)**
For each flagged section, dispatch a subagent that reads the section content alongside the actual current code. The agent tries to falsify claims: "This section says X, but the code now does Y." Reports findings as specific line-level corrections.

### 3.4 Work-End Freshness Gate (D1, D2)

**Scope:** Module-tier classification determines check breadth:
- Changes to `api/` modules → check home repo guides AND consumer guides of dependent repos
- Changes to `runtime/` modules → check home repo guides only
- Changes to `testing/` modules → skip doc checks

**Enforcement model — dual:**
1. **Work-end hard gate (primary, LLM sessions):** After code review and before squash, the hybrid detection runs. If candidate-stale sections are found, the gate blocks until they're updated. Hard gate activation prerequisite: D4's structural anchor detection must demonstrate ≥80% precision on the initial audit validation corpus.
2. **PR-level GitHub Action (complementary, all merge paths):** Covers human PRs, manual merges, CI pipeline merges — paths that bypass work-end. Runs structural anchor check only (no LLM adversarial — too slow for CI).

**Dependent repo handling:** When api/ changes affect dependent repos, the gate creates GitHub issues on those repos with specific stale-section details. Does not block the home repo's close.

**Integration with work-end orchestrator:** New step `doc_freshness_gate` added between `impl_doc_sync` and `adr` in the orchestrator's step sequence. When hard gate is not yet activated (pre-validation), runs in advisory mode (reports findings but doesn't block).

### 3.5 Arc42Stories Refresh (D7)

Arc42stories refresh runs at **epic close** (not per-branch), using two-tier verification:

**Tier 1 — 3-check sweep (structural assertions):**
1. Issue status: `gh issue view N` for every §12 reference — remove COMPLETED issues from Active Risks
2. Class name existence: `find . -name "ClassName.java"` for every §9.4 Key files entry
3. File path validity: verify all referenced file paths still exist

**Tier 2 — LLM adversarial check (prose sections):**
For the 5 prose subsection types in §9.4 (What it adds, Key wiring, Architectural decisions, Gotchas, Pattern to replicate), dispatch an LLM agent that reads the prose alongside current code and flags inaccuracies.

Only layers modified during the epic are checked — bounded scope.

### 3.6 Dependent Repo Detection (D6)

**Primary:** Automated POM analysis via GitHub Action on casehub-parent. Runs on POM changes, writes `dependency-graph.json` to the parent repo. The work-end gate reads this cached graph — no per-close penalty.

**Supplementary:** `docs/platform/dependency-map.md` retains its Nature column (SPI signatures, runtime dep, compile scope) for change-type classification. Edge existence is automated; Nature annotations are manually maintained.

---

## 4. Initial Audit Execution (D3)

### 4.1 Slot Setup

Create a dedicated audit slot that pulls in all 28 repos. The slot enables cross-repo audit work without blocking feature branches.

### 4.2 Priority Order

Foundation repos first (highest blast radius → most downstream consumers):

| Wave | Repos | Rationale |
|------|-------|-----------|
| 1 (foundation) | platform, worker, ledger, connectors, work, qhorus, eidos, neocortex, engine, iot | API/SPI changes cascade to all consumers |
| 2 (orchestration) | ras, desiredstate, blocks, blocks-ui, claudony, openclaw, workers, ops, pages | Integration layer — consumes foundation, consumed by apps |
| 3 (application) | devtown, aml, clinical, life, drafthouse, quarkmind, soc, fsitrading, chat-app | Leaf nodes — consume platform, not consumed |

### 4.3 Per-Repo Audit Process

For each repo:

1. **Delta analysis:** `git log --since=<guide-last-updated> --oneline` — classify changes by impact (new modules, renamed types, new SPIs, removed features, architectural shifts)
2. **Source mining:** Read blog entries (workspace `blog/` + published `casehubio.github.io/_articles/`), GitHub issues, and git commit messages to reconstruct what changed and why
3. **Guide update:** Fix consumer guide, contributor guide sections affected by the delta
4. **Structural anchor insertion:** Add YAML frontmatter with anchors to each updated section (or new capability chunk if decomposing)
5. **Arc42stories check:** Run 3-check sweep on ARC42STORIES.MD if it exists
6. **Commit + close issue:** Commit updates, close the per-repo audit issue with evidence

### 4.4 Exit Criteria Per Repo

1. All consumer guide sections verified against current code by LLM adversarial check
2. All structural anchors validated via automated assertion (anchored class/SPI exists in codebase)
3. Per-repo GitHub issue closed with evidence commit

### 4.5 Validation Corpus

The first-wave audit (5 foundation repos: platform, worker, ledger, work, qhorus) produces a labeled corpus:
- **Known-stale sections** (before fixes) — true positives for staleness detection
- **Known-current sections** (after fixes) — true negatives

This corpus validates D4's structural anchor detection. If precision ≥80% (≤20% false positive rate), the work-end hard gate activates. If not, the detection methodology is refined and re-validated.

---

## 5. Capability Index Design

The cross-repo capability index (`docs/capabilities.md`) maps capabilities to documentation chunks:

```markdown
# CaseHub Capability Index

> What can I do with CaseHub? Find the capability, follow the link.

## Orchestration
| Capability | What it does | Consumer chunk | Contributor chunk |
|------------|-------------|----------------|-------------------|
| Case lifecycle | Define and execute multi-step case plans | engine/capabilities/case-lifecycle.md | engine/internals/case-execution.md |
| Work items | Human task inbox with SLA and delegation | work/capabilities/work-items.md | work/internals/work-item-store.md |
| Worker dispatch | Automated task execution and routing | worker/capabilities/worker-api.md | worker/internals/executor-pipeline.md |

## Communication
| Capability | What it does | Consumer chunk | Contributor chunk |
|...

## AI & Knowledge
...
```

Organized by domain (orchestration, communication, identity, audit, AI, UI, infrastructure, operations) rather than by repo. An LLM searching "how do I send a notification" scans the table, finds the row, and loads the specific chunk.

---

## 6. Skill Modifications

### 6.1 work-end orchestrator

New step `doc_freshness_gate`:
- **Position:** After `impl_doc_sync`, before `adr`
- **Phase:** `closing:review`
- **Type:** `judgment` (LLM decides whether findings are genuine)
- **Skip condition:** No code changes in the branch (docs-only branches skip)
- **Activation:** Advisory mode until validation corpus confirms ≥80% precision, then hard gate

### 6.2 implementation-doc-sync

Extend to check structural anchors in addition to session-scoped analysis. When a changed file is anchored by a documentation section, flag that section even if the session didn't explicitly touch it.

### 6.3 New: doc-freshness-check script

Python script called by both work-end gate and GitHub Action:
- Input: branch diff, dependency graph, guide locations
- Output: list of candidate-stale sections with evidence (which anchor changed, what the diff shows)
- Shared implementation ensures work-end and CI use the same detection logic

---

## 7. GitHub Action (PR-level enforcement)

`.github/workflows/doc-freshness.yml` on casehub-parent:
- **Trigger:** PR to main on any CaseHub repo
- **Steps:**
  1. Read `dependency-graph.json` from parent
  2. Run `doc-freshness-check.py` with PR diff
  3. If candidate-stale sections found: post PR comment listing them, set check to "action required"
  4. If no sections flagged: pass

Lightweight — structural anchor check only, no LLM adversarial. Covers the merge paths that work-end misses.

---

## 8. Risks and Mitigations

| Risk | Impact | Mitigation |
|------|--------|------------|
| Structural anchors themselves drift (class renamed but anchor not updated) | False negatives — stale section not detected | Broken anchors are mechanically detectable (class not found in codebase). CI check validates anchor integrity. |
| Work-end becomes too slow with doc gate | Developer friction, gate bypass | Structural anchor check is O(diff size), not O(repo size). LLM adversarial runs only on flagged sections. |
| Audit slot conflicts with feature work | Merge conflicts on guides | Slot is dedicated — no feature work shares the audit branches. Guide updates are additive (new content), not conflicting. |
| Monolithic guides resist decomposition | RAG quality stays poor for un-decomposed guides | Demand-driven — decompose when retrieval failures occur or guide exceeds size threshold. Not all-or-nothing. |
| Arc42stories prose drift between epics | Inaccurate architecture docs | Bounded by epic duration (1-2 weeks). Acceptable trade-off vs per-branch overhead. |

---

## References

- decisions.md — 7 validated decisions (D1-D7)
- docs/platform/dependency-map.md — current manual dependency map
- docs/INDEX.md — current documentation entry point
- docs/consumer-index.md / docs/contributor-index.md — current guide indexes
- GE-20260601-85afd0 — 3-check quality sweep for arc42stories
- GE-20260601-b0eabf — class name existence verification
- GE-20260623-e02ce2 — GitHub issue stateReason technique
- arc42stories spec §9.4 — 9 subsection types (4 structural, 5 prose)
- PLATFORM.md decomposition precedent — topic files + thin index pattern
- Blog "The Factory That Forgot" (2026-06-17) — manual merge bypass incident
