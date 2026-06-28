# HANDOFF — casehub

**Date:** 2026-06-28
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

*Updated: qhorus#309 closed — removed from backlog.*

---

## Last Session

**#315 closed — neural-text deep-dive synced with Matryoshka + quantization features from neural-text#31.**

- Updated `docs/repos/casehub-neural-text.md`: two new Key Abstractions subsections (`MatryoshkaEmbeddingModel`, `DenseQuantization`), CaseRetriever oversampling behavior, rag/ module row, LangChain4j table, C7 current-state row, ARC42STORIES.MD reference.
- Fixed 5 stale class names from #17 refactor across rag-api, rag-testing, Key Abstractions, and C7 rows (CorpusStore → EmbeddingIngestor family). Updated `EmbeddingIngestor` description to match actual interface (pre-chunked text, not documents).
- Also landed: `build-all.sh` parent POM install fix, `LIFECYCLE.md` CommitmentState `isActive()` registration.

## Immediate Next Step

Pick next work from the backlog. `soc` and `fsitrading` are ready for first domain research sessions. Or pick up a trailing item.

## What's Left

- `ledger#159` — normalize remaining event producers to dual-channel · S · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon; channel taxonomy patterns ready |
| — | casehub-soc first session | M | Med | Domain research + brainstorming |
| — | casehub-fsitrading first session | M | Med | Domain research + brainstorming |
