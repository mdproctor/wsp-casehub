# Session Handover — casehub-examples (slot 196)

**Date:** 2026-10-30
**Branch:** wip/era5-relational-eval (examples), wip/era5-relational-eval (wsp-casehub), main (neocortex)
**Era:** 5 — relational cognition, targeted somatic drives
**Issue:** #175 — Re-architect cognitive pipeline for targeted somatic drives

## Status

Golden prose corpus created as the regression target for pipeline rendering. 51 events, 5 characters, 10 situations each, all cognitive layers exercised. **Q=4.35, R=4.37, D=4.12** — beats every historical baseline.

**Four per-drive scores remain below 4.0.** These are the blocking problem. Everything else is ready for pipeline implementation once the golden prose is fully clean.

## The Blocking Problem: Villain Scheming Differentiation

### What's stuck

| Character | Drive | Q | R | D | Why it's stuck |
|-----------|-------|:-:|:-:|:-:|---------------|
| dick-dastardly | scheming | 3.62 | 3.88 | 3.38 | Scheming is HOW he does everything — it bleeds into greed, recognition, gloating events |
| hooded-claw | scheming | 4.44 | 3.89 | 4.44 | R=3.89 — register drops because scheming vocabulary is inherently cognitive ("mapping", "calculating") |
| hooded-claw | self-preservation | 3.80 | 4.20 | 4.00 | Q=3.80 — triggers 100% of events, becomes background noise, but can't be removed (identity-level) |
| hooded-claw | gloating | 4.50 | 4.50 | 3.75 | D=3.75 — blurs with scheming. Gloating vocabulary ("what they believe vs what is true") triggers scheming |

### What was tried and failed

Three separate agents attempted **trigger-rate reduction** — removing scheming/self-preservation signals from events where other drives could own the moment. All failed:

- **DD:** Removing scheming from gloating events destroyed gloating differentiation (D dropped 4.25→3.50) because DD's gloating IS about successful deception — inseparable from scheming
- **HC:** Removing self-preservation from safe events degraded Q across the board. His character voice is inherently vigilance-coded — the vocabulary carries drive signals regardless
- **PP:** Over-trigger reduction worked partially (proving-worth D 3.88→3.9) but caused protection D regression (4.17→3.7). Fixed on second pass — protection restored to 4.0 but proving-worth D still at 3.9

**Why subtraction fails for villains:** Their core drives are not independent impulses — they're MODES OF BEING. Scheming is not something hooded-claw DOES; it's HOW HE EXISTS. You can't subtract scheming from his voice without subtracting his identity.

### Angles to try (fresh session should explore BEFORE implementing)

**1. Scheming sub-types.** What if scheming has distinct somatic expressions by mode?
- *Defensive scheming* (protecting the plan) — body contracts, attention narrows to exits, time compresses
- *Offensive scheming* (advancing the plan) — body settles, attention widens to map the room, time stretches with patience
- Different body locations and channels per mode. The classifier might differentiate these from each other AND from gloating/self-preservation

**2. Genuinely non-scheming events.** Currently most events are social/strategic. What about:
- A childhood memory triggered by a smell in the kitchen (formation memory, attachment — no scheming)
- A moment of unexpected beauty (the garden at dawn — curiosity analog for villains, body responds to aesthetics)
- Physical discomfort or exhaustion (body-dominant, no room for cognitive scheming)
- A genuine emotional response the character can't control (grief, nostalgia, disgust)

**3. Per-target body-location for scheming.** Instead of one scheming location, split by target role:
- Scheming → prey: hands (precision, manipulation, delicate touch)
- Scheming → obstacle: jaw/shoulders (competitive tension, bracing)
- Scheming → tool: peripheral attention (monitoring utility, not engaging fully)
- This would make scheming feel different per target, improving D without reducing trigger rate

**4. Classifier investigation.** The classifier might have biases:
- Does it conflate scheming vocabulary with cognitive narration? "Mapping", "calculating", "angles" might trigger scheming detection even in non-scheming contexts
- The deep analysis agent found that **desc/thinking alignment matters** — the classifier reads desc for relevance, thinking for register. If the desc implies scheming but thinking is pure gloating, it scores badly
- Test: what does the classifier do with identical thinking but different desc fields?

**5. Gloating-scheming separation through temporal ordering.** The deep analysis principle: "the prickle arrived first, then something different — slower, warmer" (R=4-5) beats "warmth and prickle coexist" (R=3). For gloating events: make the scheming PRECEDE — "the plan was already done, the pieces already placed" — then gloating ARRIVES as a new sensation. Temporal ordering makes them sequential, not simultaneous.

**6. Self-preservation as a different TEXTURE, not just always-on.** HC self-preservation at 100% trigger with Q=3.80 means it's monotonous. What if it varied by context?
- In safe moments: dormant warmth, a low hum
- Under social threat: cold sharpness, neck-prickle
- Under physical threat: full armored readiness
- The drive triggers in each but FEELS different — same self-preservation, different channel selection

## What Worked — Patterns to Reuse

### Methodology: describe → test → implement

**This is now mandatory for all cognitive extensions.** The sequence:
1. Hand-craft the ideal rendered output (what the character should think/say)
2. Run the classifier — verify it scores 4.0+ on Q, R, D
3. Iterate the prose until it passes
4. THEN implement the backend to generate equivalent output
5. Iterate the backend until it matches the golden target

This was established because the reverse (implement first → discover what works) produced months of iteration on the wrong problem.

### Agent orchestration patterns

| Pattern | When it worked | When it failed |
|---------|---------------|----------------|
| One fork per character, parallel | Initial prose creation — 5 characters × 10 events in one batch | — |
| Targeted patch agents | Filling cognitive layer gaps — focused, small changes | When the patch was additive (bolted on) rather than woven in |
| Review agent after patchers | Caught metaphor collisions and voice drift | — |
| Deep analysis agent (effort:max, non-fork) | Best improvements came from this — DD and PP both +0.3-0.5 | Took 30 min but results were worth it |
| Over-trigger reduction agents | PP (marginal gain) | DD and HC (identity-level drives can't be subtracted) |
| Send-back for regressions | PP protection restored after initial fix caused regression | — |

### Scoring principles from deep analysis

1. **Body-location differentiation beats vocabulary differentiation.** Give each drive a unique body LOCATION — greed=gut-deep, recognition=chest-high, scheming=hands/periphery, gloating=central-spreading. When two drives share a location, D drops.

2. **Trigger-rate reduction improves differentiation** (for independent drives only). A drive at 60% trigger with D=4-5 beats 90% trigger with D=3. Does NOT work for identity-level drives.

3. **Desc/thinking alignment is essential.** The classifier reads desc for drive relevance, thinking for register. Misalignment penalises both.

4. **Temporal discovery ordering.** "The prickle arrived first, then something different — slower, warmer" (R=4-5) beats "warmth and prickle coexist" (R=3). Sequential discovery over simultaneous description.

5. **Drives peak when owning events.** Pure savoring (one drive, whole event) scores R=5. Same drive competing with 3 others in one event scores R=3.

### Barrett 4-channel selection framework

**Not a checklist — a selection space.** Each drive×role×situation selects 1-2 channels, not all 4.

**Drive → primary/secondary channel:**

| Drive | Primary | Secondary |
|-------|---------|-----------|
| scheming | attention | action-readiness |
| suspicion | attention | body |
| curiosity | attention | action-readiness |
| protection | action-readiness | attention |
| gallantry | action-readiness | body |
| gloating | body | time |
| dominance | body | attention |
| self-preservation | body | attention |
| loyalty | body | action-readiness |
| social-harmony | body | attention |
| greed | body | attention |
| recognition | body | time |
| proving-worth | body | time |
| adventure | body | time |

**Target role → channel boost:**

| Role | Boosts |
|------|--------|
| prey | attention |
| obstacle | action-readiness |
| threat | body + attention |
| ward/beloved | body + action-readiness |
| tool | attention |
| rival | action-readiness + time |
| ally | body |

These are mechanical lookup tables — no LLM needed for the selection. The LLM generates the prose within the selected channels during consolidation.

## Architecture Decisions (locked in)

### Drive-first rendering

```
DRIVE (intensity) → Target Name
[Barrett channel-selected description — 1-2 channels most relevant to drive×role×situation]
```

Each drive appears once per relevant target. The Barrett description is a single coherent paragraph, not a 4-point checklist.

### Relational information integrates INTO drives, not alongside

Currently 5 separate sections reference each nearby person (Social Awareness, Theory of Mind, Relational Schemas, Trust, CognitionCore). All collapse into the drive-first structure:
- Trust → shapes channel intensity (trust=low → body tightens, attention sharpens)
- Theory of Mind → empathic sensing woven into drive expression ("you sense their distress")
- Social Awareness → becomes felt comparison, not analytical label
- Relational Schema → provides the ROLE that selects which channels foreground

### Mechanical channel selection

The per-tick pipeline is almost entirely mechanical:
1. Who's nearby? → world state lookup
2. Their roles? → schema overlay lookup
3. Drive-role salience? → matrix lookup (computeDriveRoleSalience)
4. Filter drives? → CAPS threshold arithmetic
5. Select target per drive? → max salience
6. Channel emphasis? → drive×role lookup table
7. Per-role description? → consolidation cache key lookup
8. Assemble prompt → template

Only LLM call per tick is situation classification (and character response). Appraisal, targeting, rendering are deterministic.

### All cognitive layers must be exercised and justified

The golden corpus exercises: Barrett 4-channels, beliefs, formation memories, CAPS attractors, drive differentiation, attachment, Theory of Mind, trust dynamics, relational schemas, self-standards, persona switching. Any new cognitive layer must describe its contribution to rendered output and score above threshold before implementation.

## Pipeline Implementation (pending golden prose completion)

### Step 1: Seed relational schemas into MindMap

`ManorCognitiveSeeder.seedPeople()` currently writes overlays with only PAD values. Must add:
```java
props.put("rel.role", schema.role());
props.put("rel.trust", String.valueOf(schema.trust()));
props.put("rel.intimacy", String.valueOf(schema.intimacy()));
props.put("rel.utility", String.valueOf(schema.utility()));
props.put("rel.description", schema.description());
```

Without this: P20 has nothing to evolve (0ms), P22 finds 0 schemas, per-role descriptions never generated.

### Step 2: Restructure renderCognitiveSections

Replace 5 fragmented relational sections with the drive-first integrated structure. Each cognitive section either folds into a drive or becomes a compact preamble.

### Step 3: Update DriveDescriptionGenerationPhase

Include channel emphasis hints per drive×role in the LLM prompt: "For SCHEMING toward PREY, emphasize ATTENTION and ACTION READINESS channels."

### Step 4: Eval against golden corpus

Run 30-event scenario, classify, compare against golden prose scores. Iterate pipeline until scores match.

## Eval History

| Run | Events | Q | R | D |
|-----|--------|:-:|:-:|:-:|
| **Golden prose (target)** | **51** | **4.35** | **4.37** | **4.12** |
| Era 4 baseline | 300 | 3.92 | 3.74 | 3.74 |
| Label-fix run | 30 | 3.99 | 3.68 | 3.72 |
| Era 5 targeted (with 5-arg wiring) | 30 | 3.87 | 3.46 | 3.53 |
| Era 5 + YAML briefing fallback | 30 | 3.84 | 3.27 | 3.70 |
| Era 5 full pipeline | 105 | 3.84 | 3.13 | 3.57 |

## Neocortex Issue Housekeeping

Closed 3 open issues whose work was already landed on neocortex main:
- **#714** (DriveDescriptionGenerationPhase: Barrett 4-channel profiles) — committed 5d7eaec0 + 1d520d46
- **#716** (RelationalSchema + PersonalityProfile types) — foundation consumed by #717–#720
- **#717** (Wire relational context into AppraisalContext) — 5 commits 37a7116d–f44793c7

Created: **casehubio/examples#176** — Traitors/Big Brother format evolution (parked — use if treasure hunt format can't generate enough situation variety)

## Key Files

| What | Where |
|------|-------|
| Golden prose corpus | `examples/wacky-manor/docs/eval/golden-prose-20261030/` |
| Regression gate | `golden-prose-20261030/verify_golden.py` |
| Methodology | `golden-prose-20261030/methodology.md` |
| Era 5 research | `neocortex/docs/research/era5-relational-cognition-research.md` |
| CharacterCognition | `examples/wacky-manor/src/.../agent/CharacterCognition.java` |
| CognitionObservationBuilder | `neocortex/cognition/src/.../observation/CognitionObservationBuilder.java` |
| ManorCognitiveSeeder | `examples/wacky-manor/src/.../agent/ManorCognitiveSeeder.java` |
| DriveDescriptionGenerationPhase | `neocortex/mindmap-intelligence/src/.../consolidation/DriveDescriptionGenerationPhase.java` |
| PipelineToRenderingTest | `examples/wacky-manor/src/test/.../agent/PipelineToRenderingTest.java` |
| Classifier | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| Design journal | `wsp-casehub/JOURNAL.md` |
| Consolidation cache | `~/claude/casehub/consolidation-cache/consolidation-cache-generic.json` |

## Constraints

- **Do NOT remove cognitive sections.** They represent abilities. Fix the rendering, not the cognition.
- **Do NOT accept regressions.** Golden means every dimension improves or holds.
- **Describe → test → implement.** Every cognitive extension proves its contribution to rendered output before backend work.
- **Delete consolidation cache before eval runs.** Old cache has target-agnostic descriptions.
- **Install neocortex before running examples.** Changes on neocortex main need `mvn install`.
- **No pipeline work until golden prose is all-4.0+ per drive.** Fix the target first.
