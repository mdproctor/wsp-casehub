# Cognitive Research Content Inventory

Audit date: 2026-10-23
Issue: casehubio/examples#144

Complete inventory of cognitive architecture research content across all CaseHub repos,
with cross-reference against era journals to identify what's genuinely un-consolidated.

**Follow-up issues filed:**
- examples#145 — Era2 gap: 7 un-consolidated blog entries from issue-115 series
- examples#146 — Pre-era and cross-repo cognitive research consolidation

---

## Cross-Reference Summary — What's NOT In Any Era

### Un-consolidated findings with unique citations or empirical data

These contain findings, literature, or measurements that would be **lost** if not consolidated.

#### Era2-adjacent (drive expression pipeline)

| # | Source | Location | Key unique content |
|---|---|---|---|
| 1 | Wiring the Missing Connections (2026-10-09) | wsp-casehub branch issue-115 | Theoretical foundation: CAPS/Scherer/Klinger integration, three wrong turns, Read/Miller Virtual Personalities, Riddell/Wolf/Lappe 2026. ~5000 words |
| 2 | When Characters Stop (2026-10-13, Part 1) | wsp-casehub branch issue-115 | Problem statement: prescriptive→pipeline mean 3.96→3.70, five drive collapses, data-tuning trap, ACL 2025 persona research |
| 3 | From Authoring to Architecture (2026-10-13, Part 2) | wsp-casehub branch issue-115 | Conway SMS, Singer SDMs, Klinger current concerns, `serves` annotations, micro test revelation (LLM CAN express at 5/5 — gap is routing) |
| 4 | When Drives Compete (2026-10-14, Part 3) | wsp-casehub branch issue-115 | Metacognitive monitoring: Dimaggio/Lysaker, Semerari MAS, Barrett emotional granularity. PP gallantry 3→5/5. The KEY novel finding |
| 5 | Teaching LLM Drive Uniqueness (2026-10-17, Part 4) | wsp-casehub branch issue-115 | Barrett constructed emotion for discrimination, category concentration, substrate-vs-mechanism distinction |
| 6 | Classifier Couldn't See Triumph (2026-10-18, Part 5) | wsp-casehub branch issue-115 | Encoding gap detail, Barrett construct theory for encoding, Penelope delete-before-write bug, gloating 2.37→3.56 |
| 7 | Discriminative Drive Profiles (2026-10-16) | wsp-casehub main | Contrastive prompting methodology, 13/13 cross-matrix, 4/8 pipeline, kitchen-sink→discriminative |

#### Era3-adjacent (cognitive layering + consolidation)

| # | Source | Location | Key unique content |
|---|---|---|---|
| 8 | Multi-channel Drive Expression (2026-10-22) | wsp-casehub branch issue-122 | somatic-only=2.16 vs multi-channel=3.97. Damasio critique (Dunn et al. 2006), Scherer CPM, Barrett 2017, Frijda 1986. 6 paper-ready citations |
| 9 | Consolidation Trigger Decay (2026-10-22) | wsp-casehub main | Full literature: Park et al. 2023, Shinde 2026 SCM, CLS 2016. Era3 P80 has one sentence; blog has complete investigation |
| 10 | Monotonic Consolidation Trigger (2026-10-22) | wsp-casehub branch issue-661 | Implementation: per-agent vs per-tenant, dual signals (event count + divergence), observability hooks, hardening |
| 11 | Situation Channel Weighting (2026-10-23) | wsp-casehub branch issue-122 | Easterbrook empirical validation, Baumeister/Leary, Deci/Ryan. Partially in Phase 10 data, NOT in era3 |

#### Pre-era (design thinking that led to the eras)

| # | Source | Location | Key unique content |
|---|---|---|---|
| 12 | Needs Pyramid / Maslow (2026-09-19) | wsp-casehub main | 5-tier model with decay rates, drive-to-tier routing, saturation constraint, numbers-vs-words finding. Complete standalone architecture |
| 13 | Proving Emergence (2026-09-28) | wsp-casehub main | Paired state probe methodology, per-subsystem results, MoodPromptSection fix, mood-vs-beliefs finding. Partially in era1 |
| 14 | Wiring Problem (2026-09-20) | wsp-casehub main | Cross-repo cognitive wiring audit — what was disconnected and why |

#### Neocortex blog (correct repo, not in any era)

| # | Source | Location | Key unique content |
|---|---|---|---|
| 15 | CAPS Engine Design (2026-10-04) | neocortex blog | 6 psychological models as spreading activation: Bowlby, Gray BIS/BAS, Beck CBT, Rescorla-Wagner, McClelland, SDT |
| 16 | CARMA / Computational Appraisal (2026-10-05) | neocortex blog | Architecture lineage: SOAR, ACT-R, CLARION, LIDA. Scherer SEC, OCC, PAD positioning |
| 17 | Arousal-Gated Tiering (2026-10-07) | neocortex blog | Kahneman dual-process for prompt sections. HC=5/5 PP=2→4/5 calibration. PAD geometry |

#### Agency (empirical findings not in any era)

| # | Source | Location | Key unique content |
|---|---|---|---|
| 18 | Computational Emotion Research (agency issue-296) | agency specs | 745 lines. OCC model: 22 emotion types, appraisal dimensions, computational implementations. Deepest OCC literature in the corpus |
| 19 | Measurement-Driven Cognition Findings (agency issue-261) | agency specs | Full A/B data: 7-orchestrator cognitive stack vs baseline. Cognitive graph growth stats |
| 20 | When Telling AIs How to Think Makes Them Worse (2026-09-14) | agency blog | Empirical A/B: prose directives produced WORSE dialogue. Removing directives and presenting state as labeled profile data reversed result |

#### Engine

| # | Source | Location | Key unique content |
|---|---|---|---|
| 21 | Cognitive Stack Inventory (engine issue-1104) | engine specs | Complete type-system inventory of ALL cognitive types in blocks + neocortex. Reference document |

#### Examples (not yet relocated)

| # | Source | Location | Key unique content |
|---|---|---|---|
| 22 | Structured Personality Composition in LLM Agents | examples wacky-manor/docs | **Research paper draft** — 601 lines. 5-axis Eidos coordinate system, 4-layer experiment, quantitative results (turns to resolution 6.3→10.7), composition compatibility matrix. Most paper-ready document in corpus |
| 23 | LLM Autonomy Landscape 2026 | examples wacky-manor/docs | Literature survey: 9 projects (Smallville, Concordia, JPAF, OCEAN), 9-category capability taxonomy, fit-gap matrix |
| 24 | SESSION-LOG-20261007 | examples wacky-manor/docs | CLI cold start, session lifecycle, cognitive parameter findings |

---

## Already Consolidated (covered by era journals)

| Blog entry | Covered by |
|---|---|
| Ablation Taxonomy Weight (2026-10-02) | Era1 Phase 1 |
| Regression That Wasn't (2026-10-18) | Era2 Phases 4-5 |
| Social Cognition JPA (2026-09-23) | Not research — infrastructure |
| Agent That Speaks (2026-08-26) | Not cognitive research — speech synthesis |

---

## Already In Correct Location (no relocation needed)

**Neocortex blog entries** — 10+ cognitive entries already in neocortex repo. No relocation needed. Items 15-17 above are consolidation candidates for era journals, not relocation candidates.

**Parent docs/repos/** — subtree of actual repos, not copies. Canonical location.

**Agency/engine design specs** — implementation specs (goal-aware loop, drive architecture, attention mediator, obs renderers, pipeline health). Not research. Stay where they are.

---

## Already Relocated (this session)

| File | From | To |
|---|---|---|
| CHARACTER-EMERGENCE-FINDINGS.md | examples wacky-manor/docs/ | neocortex docs/research/ |
| era1-character-taxonomy-research.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| drive-expression-evaluation-research.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| era3-cognitive-layering-research.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| UNIFIED-PERSONALITY-MODEL-SYNTHESIS.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| interpersonal-relationship-models.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| schema-therapy-research-2026-10-07.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| memory-retrieval-models-research.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| psychopathy-models-research.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |
| consolidation-trigger-literature-2026-10-22.md | examples wacky-manor/docs/research/ | neocortex docs/research/ |

Phase 10 added to CHARACTER-EMERGENCE-FINDINGS with principles renumbered P85-P88.

---

## Eval Results (borderline — stay in examples)

| File | Research value |
|---|---|
| eval/multichannel-pipeline-2026-10-14/RESULTS.md | Barrett/Frijda/Easterbrook citations, pipeline fix narrative |
| eval/caps-descriptions-2026-10-17/RESULTS.md | Mischel/Shoda citation, CAPS semantic fix |
| eval/appraisal-filtering-2026-10-15/RESULTS.md | Pre/post drive discrimination data |
| eval/tuned-run-2026-10-22/PARAMETER-TUNING.md | Session pool, accumulator, mood decay tuning |

These are app-specific eval run outputs. The research findings they contain should be captured in the era journals that reference them. The raw data stays in examples.

---

## Counts

| Category | Count |
|---|---|
| Un-consolidated (unique findings/citations) | 24 |
| Already consolidated (covered by eras) | 4 |
| Already in correct location | 10+ |
| Already relocated this session | 10 |
| Eval results (stay in examples) | 4 |
| Design specs (not research, stay) | ~12 |
| **Total documents audited** | **~57** |
