# Handoff — 2026-06-04

**Head commit (project):** 244d447 — docs(#238): qhorus-channel-dual-identity protocol
**Head commit (workspace):** 380fc0a — docs: add blog entry 2026-06-04

---

## What Changed This Session

**AI Fusion direction established.** Typed fact space design written — a case-scoped epistemic workspace where Drools (typed facts), LLMs (probabilistic text), QuarkusFlow (structured data), and humans (authority) all assert with epistemic metadata. Written to `docs/specs/2026-06-03-ai-fusion-hybrid-fact-space.md`. Distinction from existing BlackboardRegistry (plan coordination) and CaseMemoryStore (text retrieval) documented.

**casehubio/neural-text bootstrapped.** Full ecosystem setup: GitHub repos, workspace, CI/CD, PLATFORM.md, deep-dive, website (SVG + project card), ARC42STORIES.MD (7 epics, 7 layers, C4 diagrams), 9 Maven modules with all pom.xml files and source dirs. Ready for Java files.

**Gap issues filed.** #154 (AI observability), #155 (policy engine), #156 (eval), #157 (artifact pipeline), #158 (casehubio/neural-text onnx inference), #164 (casehub-rag), #158 updated and Hortora/spec#15 updated after Hortora corrected the RAG scope.

**write-content skill updated.** README form added (`[R]`) with section-to-mode map and "stripping trap" rule. Installed via sync-local. cc-praxis on branch `issue-110-router-skills`.

**6 doc-sync issues closed.** #144 (already fixed), #145, #146, #147, #149, #150 — all merged via PRs #151, #152.

---

## Immediate Next Step

Start neural-text implementation: Epic 2 — native image prototype. ONNX Runtime JNI + HuggingFace Tokenizers JNI in Quarkus native image on macOS ARM. This gates everything else.

---

## What's Left

- `clinical`, `devtown`, `drafthouse` — upstream delivery still outstanding from repo alignment audit · XS · Low
- Workspace branch cleanup — `issue-12` through `issue-19` deletion (overdue) · XS · Low
- `engine`, `work`, `qhorus`, `ledger`, `flow` — diverged repos, each needs own session · varies · High
- `platform`, `quarkmind` — add upstream remote · XS · Low
- cc-praxis `issue-110-router-skills` branch not merged — README form needs PR and merge · XS · Low

---


## Key References

- AI Fusion spec: `casehubio/parent: docs/specs/2026-06-03-ai-fusion-hybrid-fact-space.md`
- ONNX inference brief: `casehubio/parent: docs/specs/2026-06-03-standalone-rag-retrieval-brief.md`
- ARC42STORIES.MD: `casehubio/neural-text: ARC42STORIES.MD`
- Blog: `blog/2026-06-04-mdp01-ai-fusion-and-neural-text.md`
- Hortora/spec#15 — neural-text alignment (Hortora's design spec is authoritative for inference-* modules)
