---
title: Chunking the Platform for LLMs
date: 2026-07-07
slug: chunking-the-platform-for-llms
entry_type: diary
projects: [casehub-parent]
---

PLATFORM.md was 685 lines. That's not huge by document standards, but it's the wrong shape for how LLM sessions actually consume it.

An LLM working in casehub-aml loads PLATFORM.md at session start, looking for coherence context. Maybe it needs the Flyway conventions — 30 lines buried in a 685-line file. Maybe it needs to know whether a notification system exists. It reads the whole thing, most of it irrelevant to the task. Worse: the notification system isn't even in PLATFORM.md, because it shipped after the last doc update.

That's two distinct problems. The first is relevance density — the ratio of useful content to total content loaded. The second is coverage — capabilities that exist in code but not in documentation. Both produce the same outcome: the LLM makes decisions without the information it needs.

I'd been watching this compound across repos. An LLM in AML built a custom work-item-inbox component. blocks-ui already had one — 14 shared components, in fact — but nothing in PLATFORM.md or AML's CLAUDE.md pointed to them. The pages web component framework had no deep-dive at all. The routing architecture spans four repos (ledger computes scores, blocks owns policy config and AI strategies, engine runs classical strategies) and was documented nowhere as a coherent system.

The fix was structural, not incremental. We decomposed PLATFORM.md into 17 topic-scoped chunks — each 100-300 lines, each with a scope header that lets an LLM bail in the first 5 lines if it loaded the wrong one. A discovery index (INDEX.md, 154 lines) organises everything by concern, not by repo. Two audience guides — one for app builders, one for platform builders — provide the navigation that prevents blind spots.

The app-builder guide is where the real value sits. It has a capability matrix showing what blocks and blocks-ui provide, which apps implemented which capabilities, and at which layer. When an LLM starts building SOC, it reads the matrix and knows: AML is the closest template (regulated domain, compliance, oversight, trust routing). It sees the 14 shared UI components. It finds the pattern catalogue showing how AML and Clinical each handled trust routing, oversight gates, web UI composition. The cross-app learning happens through the document structure, not through the LLM independently discovering each repo's deep-dive.

The restructuring surfaced some design questions worth recording. I'd originally defaulted to conservative extraction — don't move patterns to blocks until two apps need them. That's wrong for LLM-driven development. If a pattern lives in an app repo, an LLM in another repo will never find it. Being in a shared location is what enables discovery, and discovery is what enables reuse. The extraction test isn't "do two apps need this?" — it's "could another app plausibly use this?" But the placement still matters: blocks owns AI/LLM patterns and cross-foundation compositions, platform owns generic utilities, engine owns execution strategies with runtime dependencies. The blocks CLAUDE.md has explicit scope criteria that the guide now references.

The protocol integration was another decision point. Protocols live in the garden with their own index. The temptation was to summarise key rules in each topic chunk — put the Flyway migration rules inline in persistence.md. We went with link-only: a one-line scope statement and a link to the authoritative protocol text. Summaries drift from their source. The same principle that keeps the audience guides free of substantive content applies here — one source of truth per rule, everything else is a pointer.

The adversarial review caught several blind spots in the design itself. The Platform Coherence Protocol (the 6-step pre-implementation checklist) had no assigned home in the new structure. The Capability Ownership table — the "where does X live?" lookup — was missing. The spec originally claimed PLATFORM.md was 2000 lines; it's 685. The motivation held (relevance density, not raw line count) but the framing needed correction.

The staleness question is the one I'm least confident about. We added chunk ownership headers and extended the doc-sync step to include chunk freshness checks. Whether that's enough to prevent the same drift from recurring depends on whether sessions actually read and notice stale content. The mechanism is sound — it's the same one that keeps deep-dives current today — but the surface area is larger now (17 chunks vs 1 monolith). We'll find out.

All 27 deep-dives got verified or updated in the same pass. The neocortex deep-dive still referenced CorpusStore (renamed to EmbeddingIngestor months ago). The blocks deep-dive said "scaffold" despite having six production packages with real consumers. The connectors deep-dive missed the entire ChatPlatform SPI and RichCard model. Sixteen CLAUDE.md files across repos now point to INDEX.md and the appropriate audience guide instead of the monolithic PLATFORM.md.

The total change: 55 files, ~4800 insertions. About 30% reorganised existing content, 70% new documentation for capabilities that had outgrown their coverage. That ratio tells you something about where the real problem was — not structure, but gaps.
