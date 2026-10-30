# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-30
**Branch:** wip/era5-relational-eval (examples), wip/era5-relational-eval (wsp-casehub), main (neocortex)
**Era:** 5 — relational cognition, targeted somatic drives
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## Status

Pipeline infrastructure complete (Steps 1-5). Golden prose corpus is the verified target. P22 prompt VERIFIED to produce golden-prose-quality descriptions (contrast formulas, discovery verbs, body-location territories). Rendering layer correctly wraps them with bold anchors and paragraph structure.

**Current gap:** The LLM ignores the structural guidance and writes analytical prose instead of somatic prose. The prompt is right — the LLM compliance is the remaining problem. Two eval runs show steady improvement (R: 3.13→3.33→3.47) but still far from golden (4.20).

**Next step:** Iterate on ONE smoke test until the LLM output matches golden prose event 2. This is experimental prompt engineering — tweaking the Barrett instruction, section ordering, and description format until the LLM follows the somatic structure. See "How to Iterate" below.

## How to Iterate (for next session)

The approach is NOT full scenario eval runs. It's focused, one-event-at-a-time comparison:

### 1. The Smoke Test

Run `PipelineToRenderingTest#goldenScenario_hcEntranceHall_fullPromptDump`. This test:
- Reproduces golden prose event 2 conditions (HC in entrance hall, all characters present)
- Seeds all 4 HC drives with golden-prose-quality P22 descriptions (contrast, discovery verbs)
- Dumps the COMPLETE prompt the LLM receives — every section, every drive description
- Runs in ~5 seconds (no LLM call)

```bash
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml test -Pfast -Dtest="PipelineToRenderingTest#goldenScenario*" -s .mvn/slot-settings.xml
```

### 2. Send the Prompt to the LLM

Take the dumped prompt and send it to Haiku 4.5 via Vertex API. Compare the thinking output against the golden prose target (event 2 in `golden-prose-20261030/transcript.json`, character "hooded-claw", second event with thinking).

### 3. Compare Line by Line

The golden prose event 2 thinking has THREE structural elements the pipeline must reproduce:
1. **Bold entity anchors**: `**Clara signs the guest book**`, `**Behind my sternum**`, `**Hartwell hovers near Clara**`
2. **Body-location paragraphs**: each paragraph grounded in anatomy (sternum warmth, spine vertical, ears tracking)
3. **Contrast text**: "not the spine's territorial claim, not the attention's angular mapping, but the slow private sweetness"

The v2 pipeline output LACKS all three. The LLM writes: "Dusty, grand, full of people I need to manage" — analytical planning, not somatic discovery.

### 4. What to Tweak

The pipeline components that affect LLM output (in order of likely impact):

| Component | File | What It Controls |
|-----------|------|-----------------|
| **Barrett instruction** | `CharacterCognition.java:208-242` | "How to Feel" section — tells LLM HOW to write thinking |
| **Drive descriptions** | Seeded in smoke test (from P22) | "What Pulls at You" — the somatic palette |
| **Section ordering** | `CharacterCognition.renderCognitiveSections()` | Which sections come first/last |
| **P22 system prompt** | `DriveDescriptionGenerationPhase.java:43-66` | How P22 generates descriptions |
| **Body-location table** | `CognitionObservationBuilder.DRIVE_BODY_LOCATIONS` | Anchor text per drive |

The Barrett instruction is the most likely fix — it needs to be MORE DIRECTIVE about using body-location anchors. Currently "give each active pull its own paragraph" is a suggestion. It might need to be: "Your aside MUST be structured as paragraphs starting with bold body locations from the drives above."

### 5. Iterate

Change one thing → re-run smoke test → send prompt to LLM → compare → repeat. Each iteration is ~30 seconds (5s test + 25s LLM call). NOT a full scenario eval.

Only run full eval (classify_emotions.py) AFTER the single-event output matches golden prose patterns.

## What Was Accomplished This Session (session 2 — pipeline implementation)

### Pipeline Steps 1-5 complete

| Step | What | Status |
|------|------|--------|
| 0 | Entity integration decision | Deferred — filed neocortex#732, #733 |
| 1 | Seed relational schemas into MindMap | Done — overlay nodes + cognitive nodes + subgraph name fix |
| 2 | Drive-first rendering with body-location anchors | Done — `DRIVE_BODY_LOCATIONS` in CognitionObservationBuilder |
| 3 | Body-location constraints in P22 | Done — body territory, channel emphasis, contrast context |
| 4 | Structural rendering in Barrett instruction | Done — paragraph structure, discovery verbs, woven clauses |
| 5 | Eval against golden corpus | Done — v1 and v2 evals, golden scenario smoke test |

### Key findings

1. **P22 generates golden-prose-quality descriptions** when given the updated prompt (verified via direct Vertex API call). The system prompt now includes: body territory constraint, explicit contrast formula, discovery verbs, 3-5 sentences.

2. **The rendering layer correctly wraps descriptions** with bold body-location anchors (`**sternum, breastbone** — GLOATING`), paragraph separation, and contrast text.

3. **The LLM doesn't follow structural guidance.** Despite the Barrett instruction saying "give each active pull its own paragraph starting with its bold body-location anchor", the LLM writes analytical prose ("full of people I need to manage") instead of somatic prose ("chest settles into architectural stillness"). This is the core remaining problem.

4. **Register recovering steadily:** R went from 3.13 → 3.33 → 3.47 across two iterations. The structural changes ARE helping, but the LLM compliance gap remains.

### Fixes made along the way

- ClassificationResult API migration (4 test files — caps-api changed `classify()` return type)
- BulkReExtractionService CDI exclusion (new from upstream #730)
- GENERIC_ID_MAP in classifier (maps renamed character IDs for eval)
- 657 tests pass (16s fast suite)

### Previous session: The differentiation problem (resolved)

Five per-drive dimensions were below 4.0 at session start:

| Target | Before | After | Technique |
|--------|:------:|:-----:|-----------|
| DD scheming Q | 3.62 | 4.25 | Body-location separation + explicit contrast |
| DD scheming R | 3.88 | 4.40 | Body-location separation |
| DD scheming D | 3.38 | 4.50 | Explicit contrast formula ("not X, not Y, but Z") |
| HC scheming R | 3.89 | 4.00 | Somatic discovery replacing cognitive calculation |
| HC self-pres Q | 3.80 | 4.00 | Woven thread (subordinate clauses in every paragraph) |
| HC gloating D | 3.75 | 4.50 | Paragraph structure + bold body-location anchors |
| PP proving-worth D | 3.88 | 4.10 | Body-location separation from gallantry/protection |

### Six rendering techniques discovered

Documented in full in `neocortex/docs/research/era5-relational-cognition-research.md` §Golden Prose Rendering Methodology. Summary:

1. **Body-location separation** — each drive owns a unique anatomical territory (scheming=fingertips/periphery, gloating=sternum, dominance=spine/jaw/feet, etc.). When two drives share a location, D drops.

2. **Explicit contrast formula** — "Not [drive-A channel]. Not [drive-B channel]. Something utterly different: [this drive's expression]." Three negations → D=5. Two → D=4. Zero → D=3. The single most powerful differentiator.

3. **Paragraph-based drive separation** — structural `\n\n` breaks + bold body-location anchors (`**Behind my sternum, warmth pools**`) give the classifier drive boundaries. Turned HC gloating D from 3.75 to 5.0.

4. **Woven thread pattern** — pervasive drives (self-preservation) expressed as subordinate clauses within other drives' paragraphs, not in separate sections. Matches cognitive architecture: HC is always monitoring. Resolved the gloating-D vs self-pres-Q tradeoff.

5. **Desc/thinking alignment** — desc signals relevance, thinking drives Q/D/R. Misalignment penalises both.

6. **Somatic discovery** — "my weight shifts" (R=4) beats "the question is whether" (R=3). Discovery verbs (arrives, settles, pools, fizzes) replace analytical verbs (calculates, assesses, determines).

### Classifier noise finding

Haiku 4.5 has ±0.2 noise per drive dimension per run on identical prose. Drives at 4.0 boundary oscillate 3.9-4.1. This was confirmed by multiple re-runs on unchanged transcripts. Pipeline eval should target means of 4.2+ to survive noise, or average multiple runs.

## Upstream Neocortex Capabilities (now available)

The slot's neocortex main was rebased onto canonical main. 22 commits spanning 6 issues are now in the history beneath the era5 work. Three are directly relevant to pipeline implementation:

### #721 — Entity Annotations (10 commits)

Full entity reference resolution system:
- **annotation-api** — SPI, value types, span resolver, anchor parser
- **annotation-sqlite/inmem** — persistence with Flyway migrations, in-memory for tests
- **MindMapExtractor** extended with entity mention spans
- **Extraction pipeline** wired to produce entity annotations from memories
- **Entity-aware salience** — retrieval modulated by entity relevance
- **Entity-scoped reflection** — belief revision and Theory of Mind per entity
- **Biography anchor syntax** — entity references in memory text resolved to specific people

**Relevance to pipeline:** The golden prose implicitly exercises entity knowledge — HC references Clara's trust, Hartwell's competence, Danny's suspicion. #721 provides the infrastructure to resolve this knowledge SYSTEMATICALLY per-tick rather than relying on the LLM to generate it ad hoc. Entity-scoped memories could enrich drive descriptions with specific, grounded knowledge about each person.

### #731 — Cognitive Pipeline Wiring (1 commit)

Reconnects three features lost during blocks→agency rename:
- **CognitiveProfileParticipant** at TERMINAL — renders EntityKnowledgePromptSection (what the agent knows about entities in conversation) and SocialComparisonPromptSection (how multiple agents perceive the same entity)
- **DomainActivationParticipant** at TERMINAL — renders DomainActivationPromptSection (cross-domain emotional correlations via DTW on PAD time series)
- All gate on CognitionConfig flags (already true in `CognitionConfig.all()`)

**Relevance to pipeline:** These three sections add cognitive content to the prompt. The era5 attention competition finding showed that separate cognitive sections compete with somatic register. Decision needed: should EntityKnowledge/SocialComparison/DomainActivation be rendered as SEPARATE sections (attention competition risk) or FOLDED INTO drive paragraphs (matching the "relational information folds INTO drives" architecture decision)?

### #727 — Cognitive Observability API (2 commits)

GraphMutation types extracted to tier-1 module. Foundation for runtime observation of cognitive state changes.

### Decision Point: Entity Integration First?

**Key question for next session:** Should the pipeline implementation START by integrating #721 entity annotations into the rendering methodology, BEFORE restructuring the prompt sections?

Arguments for entity-first:
- Entity knowledge feeds into what each drive paragraph can reference about specific people
- Per-entity ToM and memories could make drive descriptions more specific and grounded (matching golden prose)
- The #731 sections (EntityKnowledge, SocialComparison) need a decision about folding vs separate — better to make this decision with full understanding of what entity data is available

Arguments for render-first:
- The golden prose target is already defined — get the rendering structure right first, then enrich with entity data
- Entity integration adds complexity that might delay the first eval comparison

## Pipeline Implementation Steps

Revised to account for upstream capabilities:

### Step 0 (new): Assess entity integration

Review #721 entity-scoped reflection, #731 prompt sections. Decide:
- Does EntityKnowledgePromptSection fold into drive paragraphs or stay separate?
- Does entity-aware salience feed into drive description generation?
- Does per-entity ToM enrich the relational schema used for drive targeting?

### Step 1: Seed relational schemas into MindMap

`ManorCognitiveSeeder.seedPeople()` → add `rel.role`, `rel.trust`, `rel.intimacy`, `rel.utility`, `rel.description`. Without this: P20 evolves nothing, P22 finds 0 schemas.

### Step 2: Restructure renderCognitiveSections

Replace fragmented relational sections with drive-first paragraph blocks. Integrate entity knowledge into drive paragraphs if Step 0 decides fold-in. Bold body-location anchors, ordered by drive hierarchy, pervasive drives as woven threads.

### Step 3: Update DriveDescriptionGenerationPhase

- Body-location constraints per drive (lookup table)
- Contrast text generation per drive×event
- Channel emphasis hints per drive×role
- Pervasive-drive threading template (trigger rate ≥90%)
- Entity-specific enrichment if Step 0 decides integration

### Step 4: Add structural rendering to thinking field

- Paragraph breaks (`\n\n`) between drive sections
- Bold body-location anchors at paragraph starts
- Discovery verbs enforced in Barrett instruction
- Woven subordinate clauses for pervasive drives

### Step 5: Eval against golden corpus

Run 30-event scenario, classify, compare per-drive means against golden scores. Target: within 0.2 of golden. Iterate pipeline until scores match. Then 300-tick validation.

## Architecture Decisions (locked in)

### Drive-first rendering with structure

```
**[body-location anchor]** — [Barrett channel-selected description].
Not [other-drive-channel]. Not [other-drive-channel].
[Unique expression] — [pervasive-drive subordinate clause].
```

### Drive → body-location table (mechanical, no LLM)

| Drive | Body Location | Channel |
|-------|--------------|---------|
| scheming | fingertips, periphery, visual field | attention + action-readiness |
| gloating | sternum, breastbone | body + time |
| greed | gut, navel, ribs | body + attention |
| recognition | upper chest, spine | body + time |
| dominance | spine, jaw, feet | body + attention |
| self-preservation | ears, hearing, peripheral awareness | attention + body |
| protection | legs, neck, positioning | action-readiness + attention |
| gallantry | hands (offered), chest (warmth) | action-readiness + body |
| proving-worth | shoulders (weight), jaw (clenching), grip | body + time |
| loyalty | chest (deep ache), alignment | body + action-readiness |
| suspicion | neck (prickling), gut (knowing) | attention + body |
| curiosity | fingertips (tracing), chest (fizzing) | attention + action-readiness |
| social-harmony | chest (settling), jaw (softening) | body + attention |

### Drive type classification (for rendering pattern)

| Type | Rendering Pattern | Criterion |
|------|------------------|-----------|
| Discrete | Own paragraph with bold anchor | Trigger rate < 90% |
| Pervasive | Woven subordinate clauses | Trigger rate ≥ 90% |

### Relational information folds INTO drives

Trust, ToM, Social Awareness, Relational Schema all collapse into drive paragraphs. No separate cognitive-analytical sections.

### Mechanical channel selection

Per-tick pipeline is almost entirely mechanical. Only LLM call is situation classification (and character response). Appraisal, targeting, rendering are deterministic.

## Eval History

| Run | Events | Q | R | D | Notes |
|-----|--------|:-:|:-:|:-:|-------|
| **Golden v12 (target)** | **51** | **4.25** | **4.20** | **4.20** | All per-drive targets ≥4.0 |
| Golden original | 51 | 4.35 | 4.37 | 4.12 | 5 per-drive targets below 4.0 |
| Era 4 baseline | 300 | 3.92 | 3.74 | 3.74 | — |
| Era 5 targeted (5-arg) | 30 | 3.87 | 3.46 | 3.53 | — |
| Era 5 full pipeline | 105 | 3.84 | 3.13 | 3.57 | — |
| Era 5 pipeline v1 | 350 | 3.83 | 3.33 | 3.66 | Body-location anchors, channel emphasis, structural rendering |
| **Era 5 pipeline v2** | **858** | **3.86** | **3.47** | **3.72** | Improved P22 prompt: contrast, discovery verbs, 3-5 sentences |

## Key Files

### Iteration files (the ones you need for the next session)

| What | Where |
|------|-------|
| **Golden scenario smoke test** | `examples/wacky-manor/src/test/java/.../agent/PipelineToRenderingTest.java` → `goldenScenario_hcEntranceHall_fullPromptDump` |
| **Golden prose corpus** | `examples/wacky-manor/docs/eval/golden-prose-20261030/transcript.json` — event 2 (aside) for "hooded-claw" is the target |
| **Barrett instruction** | `examples/wacky-manor/src/.../agent/CharacterCognition.java` lines 208-242 — "How to Feel" section |
| **P22 system prompt** | `neocortex/mindmap-intelligence/src/.../consolidation/DriveDescriptionGenerationPhase.java` lines 43-66 |
| **Body-location table** | `neocortex/cognition/src/.../observation/CognitionObservationBuilder.java` → `DRIVE_BODY_LOCATIONS` |
| **Drive rendering** | `neocortex/cognition/src/.../observation/CognitionObservationBuilder.java` → `renderInternalState()` |
| **Social config (GENERIC)** | `examples/wacky-manor/src/main/resources/META-INF/eidos/social-config-generic.yaml` |

### Infrastructure files

| What | Where |
|------|-------|
| Classifier | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| Pipeline v1 eval | `examples/wacky-manor/docs/eval/era5-pipeline-v1/full-eval.json` |
| Pipeline v2 eval | `examples/wacky-manor/docs/eval/era5-pipeline-v2/full-eval.json` |
| ManorCognitiveSeeder | `examples/wacky-manor/src/.../agent/ManorCognitiveSeeder.java` |
| DriveDescriptionGenerationPhase | `neocortex/mindmap-intelligence/src/.../consolidation/DriveDescriptionGenerationPhase.java` |
| CognitionObservationBuilder | `neocortex/cognition/src/.../observation/CognitionObservationBuilder.java` |
| Consolidation cache | `~/claude/casehub/consolidation-cache/consolidation-cache-generic.json` |
| Era 5 research | `neocortex/docs/research/era5-relational-cognition-research.md` |
| Design journal | `wsp-casehub/JOURNAL.md` |

## Constraints

- **Do NOT remove cognitive sections.** They represent abilities. Fix the rendering, not the cognition.
- **Do NOT accept regressions.** Golden means every dimension improves or holds.
- **Describe → test → implement.** Every cognitive extension proves its contribution to rendered output before backend work.
- **Delete consolidation cache before eval runs.** Old cache has target-agnostic descriptions.
- **Install neocortex before running examples.** Changes on neocortex main need `mvn install`.
- **Isolation principle.** Change one thing at a time, measure, keep or revert. Combined changes cause unattributable regressions (v6 lesson).
- **Structure matches cognitive architecture.** Discrete drives get paragraphs. Pervasive drives get woven threads. The rendering pattern must match how the drive actually operates.
