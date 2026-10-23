# Cognitive Research Content Inventory

Audit date: 2026-10-23
Issue: casehubio/examples#144

Complete inventory of cognitive architecture research content across all CaseHub repos.
Organised by current location, with content type and key metadata.

---

## examples (wacky-manor/docs/)

### Research journals and findings

| File | Lines | Type | Citations | Empirical | Principles |
|---|---|---|---|---|---|
| `CHARACTER-EMERGENCE-FINDINGS.md` | 1341 | Master findings doc | Lazarus, Scherer, Frijda, Easterbrook, Barrett, Mischel/Shoda | Yes — 9 phases of eval data | P1-P78 (P85-P88 on issue-122 branch) |
| `research/era1-character-taxonomy-research.md` | 425 | Era 1 journal | — | Yes — taxonomy ablation, L1 emotional echo | P1-P42 |
| `research/drive-expression-evaluation-research.md` | 221 | Era 2 journal | — | Yes — pipeline instrumentation, appraisal filter | P74-P75 |
| `research/era3-cognitive-layering-research.md` | 1048 | Era 3 journal | Scherer, Webb et al., Page, Usher/McClelland, Zacks | Yes — ToM, emotion regulation, CUSUM | P75-P84 |

### Standalone research files

| File | Lines | Type | Citations | Topic |
|---|---|---|---|---|
| `research/UNIFIED-PERSONALITY-MODEL-SYNTHESIS.md` | 800 | Cross-model synthesis | Samuel/Widiger, Simard, Bach/Bernstein, Drislane, Wiggins/Trapnell, Mehrabian, Kotov | 9 psychological models mapped to unified dimensions |
| `research/interpersonal-relationship-models.md` | 1065 | Dimensional framework | Baldwin, Kernberg, Sternberg, McCroskey/McCain, Bartholomew/Horowitz | 7 frameworks for per-relationship modeling |
| `research/schema-therapy-research-2026-10-07.md` | 1541 | Schema therapy analysis | Young et al., Lobbestael, Giesen-Bloo, Major, Simard | 18 EMSs, schema modes, Cluster B PD profiles |
| `research/memory-retrieval-models-research.md` | 84 | Memory retrieval | Bower, Park et al., Rolls, Damasio, McGaugh | Appraisal-driven memory retrieval mechanism |
| `research/psychopathy-models-research.md` | 752 | Psychopathy models | Patrick et al., Blair, Baskin-Sommers, Buckels, Hare | PCL-R, Triarchic, Dark Triad/Tetrad, VIM |
| `research/consolidation-trigger-literature-2026-10-22.md` | 66 | Literature review | Park et al., Shinde, McClelland et al., Kumaran et al., Kirkpatrick | CUSUM critique for consolidation scheduling |

### Other research-relevant docs

| File | Lines | Type | Topic |
|---|---|---|---|
| `structured-personality-composition-in-llm-agents.md` | 601 | **Research paper draft** | Full paper format — 5-axis eval, 10 judges, 4-layer experiment |
| `llm-autonomy-landscape-2026.md` | 155 | Literature survey | Park et al., Concordia, JPAF, OCEAN — competitive positioning |
| `SESSION-LOG-20261007.md` | 150 | Session findings | CLI bottleneck, session lifecycle, cognitive parameter defaults |

### Eval results with research value

| File | Type | Topic |
|---|---|---|
| `eval/multichannel-pipeline-2026-10-14/RESULTS.md` | Eval results | Pipeline fixes + somatic→4-channel switch; Barrett, Frijda, Easterbrook citations |
| `eval/caps-descriptions-2026-10-17/RESULTS.md` | Eval results | CAPS semantic description fix; Mischel/Shoda citation |
| `eval/appraisal-filtering-2026-10-15/RESULTS.md` | Eval results | Pre/post appraisal filtering drive discrimination scores |
| `eval/tuned-run-2026-10-22/PARAMETER-TUNING.md` | Eval results | Session pool contention, accumulator thresholds, mood decay rates |

---

## wsp-casehub (examples workspace blog)

### On main

| File | Lines | Citations | Empirical | Topic |
|---|---|---|---|---|
| `blog/2026-09-19-mdp01-needs-pyramid-agent-cognition.md` | — | Maslow | No | Needs hierarchy for agent fixation |
| `blog/2026-09-20-mdp01-wiring-problem-cognitive-architecture.md` | — | — | No | Cross-repo cognitive wiring audit |
| `blog/2026-09-23-mdp01-social-cognition-jpa.md` | — | — | No | Social cognition JPA |
| `blog/2026-09-28-proving-emergence.md` | — | — | Yes | Proving emergence methodology |
| `blog/2026-10-02-mdp01-ablation-taxonomy-weight.md` | — | — | Yes | Ablation taxonomy weight findings |
| `blog/2026-10-16-mdp01-discriminative-drive-profiles.md` | 101 | — | Yes — 13/13 cross-matrix, 4/8 pipeline | Contrastive prompting for profile derivation |
| `blog/2026-10-22-mdp01-consolidation-trigger-decay.md` | 74 | Park et al., Shinde, McClelland/Kumaran | Yes — zero drive adaptation with CUSUM | CUSUM wrong for consolidation |

### On branches (not merged to main)

| File | Branch | Lines | Citations | Topic |
|---|---|---|---|---|
| `blog/2026-10-09-mdp01-caps-theory-wiring.md` | issue-115 | 227 | Yes | CAPS theory wiring |
| `blog/2026-10-13-mdp01-when-characters-stop.md` | issue-115 | 59 | — | Character stopping behaviour |
| `blog/2026-10-13-mdp02-from-authoring-to-architecture.md` | issue-115 | 122 | Yes | Authoring → architecture transition |
| `blog/2026-10-14-mdp01-when-drives-compete.md` | issue-115 | 79 | Yes | Drive competition dynamics |
| `blog/2026-10-17-mdp01-teaching-llm-drive-uniqueness.md` | issue-115 | 131 | Yes | LLM drive uniqueness training |
| `blog/2026-10-18-mdp01-classifier-couldnt-see-triumph.md` | issue-115 | 106 | Yes | Classifier blind spot |
| `blog/2026-10-18-mdp02-regression-that-wasnt.md` | issue-115 | 84 | Yes | Regression was redistribution |
| `blog/2026-10-22-mdp01-multi-channel-drive-expression.md` | issue-122 | 67 | Damasio, Dunn et al., Scherer, Easterbrook, Barrett, Frijda | somatic-only=2.16 vs multi-channel=3.97 |
| `blog/2026-10-22-mdp02-monotonic-consolidation-trigger.md` | issue-661 | 81 | Yes | Monotonic consolidation trigger |
| `blog/2026-10-23-mdp01-situation-channel-weighting.md` | issue-122 | 73 | Easterbrook, Frijda, Bowlby, Baumeister/Leary | Channel weighting empirical validation |

### Specs with cognitive content

| File | Branch | Topic |
|---|---|---|
| `specs/issue-115-drive-expression-gap/` (3 docs) | issue-115 | Drive expression gap design + decisions |
| `specs/phase-d-cognitive-activation/` | common | Cognitive activation design |
| `specs/issue-126-autonomous-agent-patterns/personality-evolution-design.md` | common | PersonalityEvolution pattern |
| `specs/issue-63-fix-cdi-config-beans/drive-adaptation-design.md` | common | Drive adaptation design |

---

## neocortex (docs/blog/)

Already in correct repo — no relocation needed. Potential era3 consolidation candidates.

| File | Lines | Citations | Topic |
|---|---|---|---|
| `blog/2026-10-04-mdp01-mechanical-psychology-caps-engine.md` | — | Mischel/Shoda, Bowlby, Gray BIS/BAS, Beck CBT, Rescorla-Wagner | CAPS engine design — 6 models |
| `blog/2026-10-05-mdp01-computational-appraisal-for-llm-agents.md` | — | Scherer SEC, OCC, PAD, SOAR/ACT-R/CLARION/LIDA | Computational appraisal architecture lineage |
| `blog/2026-10-07-mdp01-when-personality-drowns-in-its-own-thoughts.md` | — | Kahneman dual-process | Arousal-gated tiering, PAD geometry, HC=5/5 PP=2→4/5 |
| `blog/2026-09-28-mdp01-when-your-agent-judges-itself.md` | — | OCC | Self-appraisal, pride/shame thresholds, compound emotions |
| `blog/2026-09-23-mdp01-how-agents-think-about-what-they-want.md` | 307 | BDI | Goal architecture |
| `blog/2026-09-23-mdp02-when-your-agent-forgets-what-it-wants.md` | — | — | Goal-affect integration, retrieval |
| `blog/2026-09-25-mdp01-teaching-agents-when-to-care.md` | — | — | Attention, consolidation, goal cognition |
| `blog/2026-09-02-mdp01-from-three-stores-to-one-mind.md` | 243 | — | Identity/personality/mindmap unification |
| Plus ~10 more touching cognitive topics | | | |

### Neocortex research (docs/research/)

| File | Topic |
|---|---|
| `2026-09-28-occ-emotion-appraisal-architecture.md` | OCC emotion appraisal (pre-existing) |
| + 10 files just relocated from examples | See examples section above |

---

## agency

### Blog entries

| File | Topic |
|---|---|
| `docs/blog/2026-08-21-mdp02-the-social-brain.md` | Social cognition architecture |
| `docs/blog/2026-08-21-mdp04-what-does-the-agent-want.md` | Drive architecture research |
| `docs/blog/2026-08-26-mdp01-the-agent-that-speaks.md` | Speech + personality |
| `docs/blog/2026-09-14-mdp01-when-telling-ais-how-to-think-makes-them-worse.md` | Measurement-driven cognition |

### Specs with cognitive research

| File | Lines | Citations | Topic |
|---|---|---|---|
| `specs/issue-296/2026-09-23-computational-emotion-research.md` | 745 | OCC model | Computational emotion research |
| `specs/issue-304/2026-09-27-cognitive-attention-mediator-design.md` | — | — | Attention signal design |
| `specs/issue-260/2026-09-14-cognitive-obs-renderers-design.md` | — | — | Cognitive observation renderers |
| `specs/issue-279/2026-09-15-cognitive-pipeline-health-design.md` | — | — | Pipeline health design |

### Research files

| File | Topic |
|---|---|
| `research/2026-08-16-autonomous-agent-patterns-landscape.md` | Autonomous agent landscape survey |

---

## wsp-casehub-neocortex (neocortex workspace)

### Plans with cognitive research

| File | Topic |
|---|---|
| `plans/attic/issue-428/cognitive-appraisal-architecture.md` | CAPS/Frijda refs |
| `plans/attic/issue-428/sec-appraisal-pipeline.md` | SEC pipeline |
| `plans/attic/issue-429/sec-pipeline-validation.md` | SEC validation |
| `plans/attic/issue-430/llm-backed-appraisal.md` | LLM appraisal strategy |
| `plans/attic/issue-407/psychology-cause-effect-models.md` | Psychology cause-effect |

---

## engine

| File | Lines | Topic |
|---|---|---|
| `docs/blog/2026-07-30-mdp01-personality-that-thinks.md` | — | Early Jungian cognitive functions |
| `docs/specs/issue-1104-hive-mind/2026-09-20-cognitive-self-improvement-research.md` | — | Cognitive self-improvement research |
| `docs/specs/issue-1104-hive-mind/2026-09-20-cognitive-stack-inventory.md` | — | Full cognitive stack inventory |

---

## parent

| File | Topic |
|---|---|
| `docs/brief/07-agent-identity-cognition.md` | Investor brief on identity/cognition |
| `docs/brief/20-drilldown-agent-identity-cognition.md` | Drilldown on identity/cognition |
| `docs/repos/casehub-neocortex/cognitive-types-guide.md` | Cognitive type reference |
| `docs/repos/casehub-neocortex/identity-memory-integration.md` | Identity-memory integration map |
| `docs/specs/issue-126-autonomous-agent-patterns/2026-08-18-personality-evolution-design.md` | PersonalityEvolution pattern |
| `docs/specs/issue-126-autonomous-agent-patterns/2026-08-20-mental-model-design.md` | BDI Theory of Mind design |
| `docs/specs/issue-126-autonomous-agent-patterns/2026-08-18-inner-life-design.md` | Inner life design |

---

## Summary by content type

| Type | Count | Locations |
|---|---|---|
| Research journals (era1/2/3 + CHARACTER-EMERGENCE-FINDINGS) | 4 | examples → neocortex (relocated) |
| Standalone literature reviews | 7 | examples → neocortex (relocated) |
| Research paper draft | 1 | examples (not yet relocated) |
| Blog entries with empirical findings | ~25 | wsp-casehub, neocortex, agency, engine |
| Design specs with cognitive research | ~12 | wsp-casehub, wsp-casehub-neocortex, agency, parent |
| Eval results with research value | 4 | examples (wacky-manor/docs/eval/) |
| Investor briefs | 2 | parent |
| Landscape surveys | 2 | examples, agency |

**Total: ~57 documents across 8 repos/workspaces**
