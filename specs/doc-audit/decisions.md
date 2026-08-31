# Doc Audit & RAG Optimization — Decisions

## D1: Work-end gate scope

**Choice:** Home repo + direct dependents
**Alternatives:**
- Home repo only — catches direct drift but misses ripple effects from API/SPI changes
- Flag dependents without auditing — cheap signal but no verification
**Rationale:** Cross-repo API/SPI changes are the primary source of guide staleness. Checking direct dependents catches the most damaging drift.
**Trade-offs:** Slower work-end (must resolve and check dependent repos). Requires dependency map to be accurate.
**Sources:** docs/platform/dependency-map.md, work-end orchestrator analysis
**Exploration:** quick
**Status:** captured

## D2: Enforcement model

**Choice:** Hard gate on home repo, GitHub issues for dependents
**Alternatives:**
- Advisory + issues for all — weaker enforcement, docs still fall behind
- Tiered (hard for API/SPI, advisory for internals) — complex to classify changes correctly
**Rationale:** Hard gate is the only way to guarantee docs never fall behind on the closing repo. Can't block cross-repo work, so dependents get issues instead.
**Trade-offs:** May slow velocity on repos with large changes. Developers must update guides before closing.
**Sources:** Current work-end gate analysis (impl_doc_sync is advisory, not enforced)
**Exploration:** quick
**Status:** captured

## D3: Fix strategy for initial audit

**Choice:** Dedicated audit slot with per-repo GitHub issues
**Alternatives:**
- Batch by staleness tier — faster but less thorough per repo
- Central sweep from parent — loses per-repo git log and blog context
**Rationale:** Each repo needs access to its own git history, issues, and blogs to reconstitute accurate guides. A slot keeps audit work isolated from feature branches.
**Trade-offs:** More sessions required. Must coordinate slot with ongoing feature work.
**Sources:** Staleness inventory (1,700+ commits across 28 repos)
**Exploration:** quick
**Status:** captured

## D4: Staleness detection methodology

**Choice:** Hybrid — diff-based triage then LLM adversarial verification
**Alternatives:**
- Diff-based only — misses semantic drift (guide says X, code does Y now)
- LLM adversarial only — slow, token-heavy, may hallucinate findings
**Rationale:** Mechanical diff scan identifies candidate stale sections (fast, deterministic). LLM agent reviews only flagged sections against actual code (targeted, semantic). Best of both: diff precision for coverage, LLM judgment for accuracy.
**Trade-offs:** Two-phase process is more complex to implement. Diff scan needs a script; adversarial check needs a subagent prompt.
**Sources:** Garden entries on arc42stories quality gates (3-check sweep)
**Exploration:** quick
**Status:** captured

## D5: RAG structure — three-layer decomposition

**Choice:** Three separate document layers — capability index, per-capability chunks, thin guide indexes
**Alternatives:**
- Capability lookup tables added to existing monolithic guides — still loads full file for one section
- Self-contained chunks only (no index) — good for retrieval, poor for session loading
**Rationale:** Separate documents avoid loading content you don't need. Capability index is the RAG entry point (small, dense). Chunks are independently loadable. Guides become thin indexes for session loading.
**Trade-offs:** Decomposing 56 existing monolithic guides into chunks is significant work. More files to maintain. Links between layers must be kept consistent.
**Sources:** Current guide sizes (200-550 lines), RAG retrieval patterns
**Exploration:** quick
**Status:** captured
**Depends on:** D4 (staleness detection verifies chunk freshness, not monolithic guides)

## D6: Dependent repo detection

**Choice:** Static dependency map from docs/platform/dependency-map.md + Maven POMs
**Alternatives:**
- Dynamic Maven analysis at gate time — always accurate but requires all repos cloned, adds ~10s per work-end
**Rationale:** Static map exists, is cheap, deterministic. Periodic audit catches map drift. Work-end gate reads it once.
**Trade-offs:** Map may be stale. Missing dependencies mean missed dependent repo checks.
**Sources:** docs/platform/dependency-map.md
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (gate scope requires dependent repo detection)

## D7: Arc42stories refresh approach

**Choice:** Extend work-end gate to cover arc42stories alongside consumer/contributor chunks
**Alternatives:**
- Separate periodic refresh cycle — decoupled from feature work but drift accumulates between sweeps
- Both gate + periodic — belt and suspenders but more maintenance
**Rationale:** The hybrid diff+adversarial check already reads git diff and dispatches adversarial checks. Adding arc42stories sections (§5 building blocks, §9 key files, §12 issues) to the check list is incremental. Garden's 3-check sweep (issue status, class names, CDI annotations) becomes a subset of the gate.
**Trade-offs:** Work-end becomes heavier. Section-level checks may miss systemic drift that spans multiple branches.
**Sources:** Garden entries GE-20260601-85afd0 (3-check sweep), GE-20260601-b0eabf (class name existence)
**Exploration:** quick
**Status:** captured
**Depends on:** D1 (gate scope), D4 (detection methodology)
