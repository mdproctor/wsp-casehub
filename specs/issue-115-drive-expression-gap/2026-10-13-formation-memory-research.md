# Formation Memory Research — Complete Reference

**Date:** 2026-10-13 (updated across multiple sessions)
**Purpose:** Internal research document for session continuity. A future session should be able to read §0 and §10-§12 and know exactly where we are and what to do next. The rest is supporting evidence — read on demand, not upfront.
**Branch:** issue-115-drive-expression-gap

---

## 0. Current State — Read This First

### What's Done
- **Phase 0 (complete):** `serves` annotation system built, all 26 formation memories annotated and audited, 6 Level 2-4 micro tests + 5 mediating substrate tests executed, two full scenario baselines established. See §7-§8 for details.
- **Root cause found (§10.1):** CAPS INPUT nodes had no vocabulary — `RuleBasedSituationClassifier` produced zero content-based situation activations. The CAPS network was personality-aware but situation-blind.
- **Theory validated (§8.1, §8.1b):** Micro tests confirm the LLM produces all drives at 5/5 on direct trigger. Cross-character tests confirm same situation → different personality → different emergent behaviour (DD gloats 5/5, PP 0/5 for identical event). If-then behavioural signatures match CAPS predictions.
- **Layer A (complete — see §11 for full results):** Weighted situation vocabulary implemented in neocortex. `CapsNode.keywords` → `CapsNode.vocabulary` (`Map<String, Double>`). All 31 INPUT nodes have construct-derived vocabulary at graded weights (strong 0.7-0.9, moderate 0.4-0.6, faint 0.2-0.3). Critical topology gap discovered and fixed: 17 INPUT nodes (including all 5 priority nodes for collapsed drives) had zero MEDIATING connections — 30 literature-sourced connections added. Topology validation test guards against regression. 33 caps-engine tests pass, 581 wacky-manor tests pass. Neocortex branch: `issue-115-weighted-situation-vocabulary`. Key empirical finding: vocabulary terms must be in their expected morphological form (inflected, not base) for exact-match classification (§11.5).

### What's Settled — Do Not Re-Derive
- **D1-D11 are MADE decisions** — see `decisions.md`. Do not re-litigate.
- **The `serves` annotation system works** — parser, loader, renderer all tested.
- **Gloating collapse is a situation-routing gap, not LLM capability or memory content** — proven by micro tests.
- **The derivation chain and all 7 breaks are mapped** (§10.4) — do not re-investigate.
- **Three-layer approach defined** (§10.7) — situation encoding, memory calibration, drive amplification. All three layers needed; redundancy not established.
- **Literature is researched** — CAPS (Mischel & Shoda), Read & Miller Virtual Personalities, Conway SMS, Singer SDMs, Klinger concerns, Scherer SECs, Festinger social comparison, Leach schadenfreude, Sacchi & Dan-Glauser personality×appraisal. Do not re-search.
- **Vocabulary is universal across characters** (Read & Miller 2002) — personality differences come from connection weights and drive amplification, not different input features. Do not create character-specific vocabularies.
- **Vocabulary weights are first approximations** — subject to empirical tuning (D4a) but derived from construct centrality, not arbitrary. The three-tier grading (strong/moderate/faint) is the right granularity.
- **All INPUT nodes must have MEDIATING connections** — a topology validation test enforces this. New INPUT nodes added in the future must include connections or the test fails.
- **Memory-to-catalogue mapping methodology is defined (§12.4)** — three tests (directional correctness, cumulative trajectory, no contradictions), OUTPUT-vs-MEDIATING rule, escalation verification. All 26 mappings completed (§12.5). Do not re-derive the methodology or the mappings.

### What's Done — Layers A + B + C + Drive Competition

All three layers and drive competition detection are complete. See §12.1 for implementation status.

**Layer A:** Weighted situation vocabulary — §11 for full results.

**Layer B:** Memory calibration — 26 memories mapped to catalogue entries (§12.5), methodology in §12.4. "What Shaped You" substrate section for all 5 characters.

**Layer C:** Drive amplification — D1 drive→CAPS injection via `DriveCompetitionDetector.DRIVE_CAPS_MAPPINGS`.

**Drive competition (§10.10):** Drives compete when they share CAPS activation nodes. Universal meta-instruction in `CognitivePreambleGenerator`: "Each drive is a distinct felt state. When multiple drives activate in the same moment, attend to which quality of feeling is at the front." PP gallantry: P0 3/5 → 4-5/5.

### What's Next

1. ~~**Literature review**~~ — **DONE** (§10.10.6). Barrett (2017), Kashdan et al. (2015), Semerari et al. (2003). Key finding: differentiation requires grounded conceptual knowledge (S6-level, formation-memory-traced), not generic metacognitive instruction (S3-level). Revised §10.10.4 architecture: classify formation memories (not drive descriptions) as primary signal.
2. **Implement inference-based drive differentiation** (§10.10.7) — `DriveProfileDerivationPhase` as a new `ConsolidationPhase`. Two layers: inferred drive CAPS profiles (replaces static `DRIVE_CAPS_MAPPINGS`), grounded differentiation text (replaces generic `DIFFERENTIATION_INSTRUCTION`). Design complete — ready for implementation.
3. **Full scenario validation** (step 3b) — AFTER inference system is built. One run against the final form. Compare to P0 (3.72) and B0 (3.96).

**Constraint:** All mechanisms must be literature-rooted, individual-specific, not hard-wired, and LLM-bias-aware (§8.5). Include redundant signals; prove redundancy empirically later — never assume it (§8.1b).

---

## 1. Core Literature Findings

### 1.1 LLM Persona Research — Demonstrations Override Instructions

**Source:** ACL 2025, Persona-Aware Contrastive Learning; Han et al. 2026 (arXiv:2602.01063)

**Finding:** When few-shot demonstrations and personality instructions conflict, the LLM follows the demonstrations. Formation memories function as few-shot demonstrations in the prompt — they are the strongest behavioral signal. Any emotional evaluation baked into a memory overrides what the personality model's trait descriptions would derive.

**Implication for our architecture:** If a memory says "that felt wrong" about a successful plan, the LLM pattern-matches to "unwitnessed success = negative" regardless of what the scheming:0.85 drive says. The memories, not the traits, determine behavior.

**Key paper:** "Simply prompting an LLM to assume a role does not guarantee the emergence of desired in-character behaviors." Persona consistency requires structural support (memory, reflection, retrieval), not just trait descriptions.

### 1.2 Conway's Self-Memory System (SMS) — Dynamic Activation

**Source:** Conway & Pleydell-Pearce (2000), Conway (2005), Conway, Singer & Tagini (2004)

**Finding:** Autobiographical memories are reconstructed, not replayed. The "working self" (current goals) modulates access to long-term memory. Episodic memories relevant to the current working self are activated through direct retrieval. This is dynamic — which memories surface depends on current motivational state.

**Implication:** Our architecture currently dumps ALL formation memories into the prompt simultaneously. In human cognition, personality FILTERS which memories are active. We need selective memory activation based on current concern state — Conway's working self mechanism.

**Maps to:** D5 (PAD-proximity memory-drive association), ConcernBasedSalienceStrategy, D6 (salience rendering strategies). These exist but formation memories aren't connected to them.

### 1.3 Singer's Self-Defining Memories (SDMs) — Structural Association

**Source:** Singer, Blagov, Berry & Oost (2013), Singer & Salovey (1993)

**Finding:** Self-defining memories have STABLE, ENDURING links to core concerns. They "possess a synchronic existence within narrative identity" and "remain relatively static." A memory about being ignored is ALWAYS linked to recognition — this doesn't change with context. The link is structural.

**Implication:** The `serves` annotation captures this structural link. Each memory declares which drive/concern it serves — this is a stable property of the memory, not a runtime computation.

**Key distinction from Conway:** Conway = which memories SURFACE (dynamic, context-dependent). Singer = which memories are LINKED to which concerns (stable, structural). Both mechanisms coexist at different levels.

### 1.4 Klinger's Current Concerns — Motivational Salience

**Source:** Klinger (1971), Klinger & Cox (2004)

**Finding:** Active motivational states ("current concerns") increase cognitive salience for concern-relevant stimuli. The effect is proportional to goal commitment. Concerns are individual-specific, not universal categories.

**Implication:** Drive type names ARE concerns in Klinger's sense. When DD's recognition drive is at 0.8 intensity, recognition-relevant stimuli (including recognition-linked memories) get higher salience. Our `ConcernBasedSalienceStrategy` partially implements this but uses keyword overlap, not structural links.

**Maps to:** ConcernBasedSalienceStrategy.java in neocortex/cognition. Currently uses keyword overlap between observation text and concern text. Formation memories aren't in the concern pool.

### 1.5 CAPS — Behavioral Signatures and If-Then Profiles

**Source:** Mischel & Shoda (1995), Read & Miller (2002), Shoda, Mischel & Wright (1994 — Wediko camp study)

**Finding:** Personality is expressed as stable if-then situation-behavior profiles, not average trait levels. Two children with identical average aggression scores had completely different behavioral signatures — one spiked when warned by authority, another when teased by peers. The signature, not the average, IS the personality.

**Implication:** We should test PATTERNS not AVERAGES. Each character should have a testable if-then profile: "in situation X, drive A expresses; in situation Y, drive B expresses." This is Level 4 in the testing pyramid.

**Maps to:** CAPS topology (caps-topology.yaml), disposition_modifiers, CapsEngine.settle()

### 1.6 Memory Function Terminology

**Source:** Pillemer (2003), Bluck (2003)

**Finding:** Autobiographical memories "serve" three functions: directive (guide behavior), self (maintain identity), social (bond with others). "Serves" is the established verb across the Pillemer, Bluck, and Singer literature for the memory-concern relationship.

**Implication:** The annotation term is `serves`, not `feeds`, `grounds`, or `addresses`. This is the literature-standard term.

### 1.7 Concern Vocabulary — Individual-Specific

**Source:** Singer (2013), Klinger (1971), McClelland (1961), Deci & Ryan (2000)

**Finding:** McClelland's three needs (achievement, affiliation, power) and SDT's three (autonomy, competence, relatedness) are too coarse for behavioral signatures. Singer and Klinger both establish concerns as individual-specific. Murray's ~20 needs are the right granularity but many didn't survive empirical testing.

**Implication:** Use drive type names as the `serves` vocabulary. They are individual-specific concerns, already mapped to CAPS nodes via D1. Higher-level themes (insecurity, need-for-validation) are derivable compositions — they emerge from the annotation graph, not from a separate vocabulary.

---

## 2. Implementation Mapping

### 2.1 What Exists

| Component | Location | Status |
|---|---|---|
| CAPS topology | `neocortex/docs/specs/2026-10-02-caps-topology.yaml` | Complete — 57 input nodes, 17 mediating, 24 output |
| Experience-to-Behaviour Catalogue | `neocortex/docs/specs/experience-behaviour-catalogue/` | Complete — 6 models (attachment, CBT, BIS/BAS, operant, trauma, Bandura), ~30 entries |
| shared_node_map | `experience-behaviour-catalogue/index.yaml` | Complete — 7 shared nodes across models |
| ConcernBasedSalienceStrategy | `neocortex/cognition/src/.../salience/ConcernBasedSalienceStrategy.java` | Implemented — keyword overlap matching |
| FormativeExperience record | `neocortex/memory-api/src/.../experience/FormativeExperience.java` | Has catalogueEntryId, situationTypes, salienceMultiplier, reinforcementSchedule, developmentalPeriod |
| FormationMemory record | `examples/wacky-manor/src/.../agent/SocialConfig.java:150` | Simple: age, episode, pleasure, arousal, dominance |
| ManorCognitiveSeeder | `examples/wacky-manor/src/.../agent/ManorCognitiveSeeder.java` | Seeds memories with PLACEHOLDER catalogueEntryId |
| ManorSocialConfigLoader | `examples/wacky-manor/src/.../agent/ManorSocialConfigLoader.java` | Parses YAML to SocialConfig records |
| NeedTier enum | `neocortex/cognition-api/src/.../need/NeedTier.java` | SAFETY, TASKS, SOCIAL, SELF_EXPRESSION, UNDERSTANDING |
| DriveAxis enum | `neocortex/cognition-api/src/.../drive/DriveAxis.java` | CURIOSITY, COMPETENCE, AFFILIATION, AUTONOMY |
| DriveExpressionEvalTest | `examples/wacky-manor/src/test/.../experiment/DriveExpressionEvalTest.java` | 6 PP proving-worth micro tests exist |
| PipelineOutputTest | `examples/wacky-manor/src/test/.../experiment/PipelineOutputTest.java` | Seeds + consolidates + dumps pipeline output |

### 2.2 Gaps — What's Not Connected

1. **FormativeExperience.catalogueEntryId** is always `"formation-memory"` — a placeholder. Should reference specific catalogue entries (e.g., `attachment-dismissive`, `bandura-mastery-efficacy`).

2. **FormativeExperience.situationTypes** is always `["formation", period]` — generic. Should include drive types from `serves` annotations (e.g., `["formation", "childhood", "scheming", "recognition"]`).

3. **FormationMemory record** has no `serves`/annotations field. Needs `List<ConcernAnnotation>` for extracted annotations.

4. **ConcernBasedSalienceStrategy** uses keyword overlap, not structural links. Formation memories aren't in the active concerns pool. The `serves` annotations would enable structural matching.

5. **No Level 0 consistency checker** exists. No static validation of memory-drive alignment.

6. **No Level 2-4 micro tests** exist for collapsed drives (HC gloating, DD gloating, PP gallantry, Penelope social-harmony, Penelope adventure). Only PP proving-worth has micro tests.

### 2.3 Drive → CAPS Node Mappings (from D1)

| Drive Type | CAPS Activations | Literature | Sourcing Tier |
|---|---|---|---|
| gallantry/prosocial | `other_safety`, `acceptance`, `approach` | Batson (1991), Penner et al. (2005) | Tier 2 |
| proving-worth/achievement | `mastery`, `competence_recognition`, `self_efficacy` | McClelland (1961) | Tier 2 |
| protection/caregiving | `other_safety`, `physical_threat` (sensitivity), `approach` | George & Solomon (2008) | Tier 2 |
| scheming/Machiavellianism | `dominance`, `strategic_cognition`, `distrust` | Christie & Geis (1970) | Tier 2 |
| gloating/schadenfreude | `dominance`, `BAS_activation` | TBD | Tier 3 |
| self-preservation | `threat_sensitivity`, `escape_assessment`, `FFFS_activation` | Gray & McNaughton (2000) | Tier 2 |
| recognition | `competence_recognition`, `self_worth`, `BAS_activation` | McClelland (1961) | Tier 2 |
| curiosity | `explore`, `BAS_activation` | Berlyne (1960) | Tier 2 |
| social-harmony | `other_safety`, `acceptance`, `proximity_seek` | SDT relatedness (Deci & Ryan 2000) | Tier 2 |
| adventure | `explore`, `BAS_activation`, `approach` | Zuckerman (1994) | Tier 2 |
| greed | `BAS_activation`, `approach`, `persist` | McClelland nAch variant | Tier 2 |
| loyalty | `other_reliability`, `proximity_seek`, `trust` | Bowlby attachment theory | Tier 2 |
| suspicion | `threat_sensitivity`, `distrust`, `BIS_activation` | Gray RST (BIS/BAS) | Tier 2 |
| dominance | `dominance`, `self_direct`, `assert` | McClelland nPow | Tier 2 |

### 2.4 Shared Node Map (from catalogue index.yaml)

```yaml
shared_node_map:
  self_worth: [attachment, cbt, bandura]
  self_efficacy: [bandura, cbt]
  other_reliability: [attachment, cbt]
  other_safety: [attachment, cbt]
  threat_sensitivity: [bis_bas, trauma]
  FFFS_activation: [bis_bas, trauma]
  BAS_activation: [bis_bas, operant]
  reinforcement_expectation: [operant, cbt, bandura]
  arousal_level: [bis_bas, trauma, attachment]
  escape_assessment: [trauma, bis_bas]
```

---

## 3. Formation Memory Audit

### 3.1 Audit Criteria

| Problem | Definition | Example |
|---|---|---|
| **Pre-resolved drive conflict** | Author decides which drive dominates: "not X, it was Y" | "The rush wasn't cruelty — it was the architecture" |
| **Contradiction** | Stated emotion conflicts with what the personality model would derive | "That felt wrong" about a successful plan (scheming:0.85 says success feels good) |
| **Action priming** | Describes what the character DOES, which the LLM pattern-matches to | "You practiced on the gardener next. Then the cook." |
| **Value declaration** | Resolves what things MEAN, pre-empting the personality model | "you stopped wanting to be liked. You wanted to be owed" |

### 3.2 Full Audit Results

#### Hooded Claw (6 memories)

| Age | Drives involved | Problems found |
|---|---|---|
| 6 | attachment, control | "Love was a transaction" — value declaration. "not because anyone wanted you" — pre-resolves meaning |
| 9 | suspicion | "people who are kind want something" — value declaration. "You were right" — validates the conclusion, primes suspiciousness |
| 12 | dominance, scheming | "The rush wasn't in getting what you wanted — it was in the control itself" — **pre-resolved conflict** (dominance over greed). "You practiced on the gardener next. Then the cook" — **action priming** |
| 14 | dominance, affiliation | "you stopped wanting to be liked. You wanted to be owed" — **pre-resolved conflict** (dominance over affiliation). **Value declaration** |
| 15 | scheming | "Not anger. Something colder. Something patient" — **pre-resolved conflict** (scheming over rage). Borderline acceptable — describes a quality without naming a drive |
| 18 | scheming | "Planning felt like breathing" — pre-resolves scheming as natural. "That trust was the most useful thing about her" — value declaration |

#### Penelope (5 memories)

| Age | Drives involved | Problems found |
|---|---|---|
| 5 | curiosity | "The search had been the real prize" — **pre-resolved** (curiosity over outcome). "You went looking for another locked thing immediately" — **action priming** |
| 7 | social-harmony | "You started telling a story about a duck" — **action priming** (conflict resolution through distraction). "You learned: if you could make people smile, the world felt safe" — **value declaration** |
| 9 | social, adventure | "before your mother told you to" — mild action priming. Generally OK |
| 11 | adventure | "not with fear but with something you'd later call 'the good kind of scared'" — **pre-resolved** (adventure over fear) |
| 13 | social-harmony, loyalty | "it wasn't a decision, more like gravity" — pre-resolves social-harmony as instinctive. "You felt lonely but not wrong" — pre-resolves the emotional balance. Borderline |

#### Peter Perfect (5 memories)

| Age | Drives involved | Problems found |
|---|---|---|
| 5 | protection, proving-worth | Father's quote: "A man is measured by what he protects, not what he wins" — direct value statement, primes protection over competition. "Look, Dad, I fixed it" — **action priming** |
| 8 | proving-worth | "warmth that had nothing to do with the grade and everything to do with having measured up" — **pre-resolved** (proving-worth over achievement) |
| 12 | protection, proving-worth | Father's quote: "That's how it works. You help, you get helped" — value instruction. "lay awake planning all the things you could learn to fix" — **action priming** |
| 14 | proving-worth | "the rush wasn't pride — it was relief" — **pre-resolved** (proving-worth over pride). "You'd proved you belonged" — pre-resolves |
| 16 | proving-worth, gallantry | "Not because she praised you — because you'd done something that mattered" — **pre-resolved** (proving-worth over recognition) |

#### Dick Dastardly (5 memories)

| Age | Drives involved | Problems found |
|---|---|---|
| 6 | greed, scheming | "the fullness wasn't just in your stomach. It was in knowing you'd solved the problem yourself" — **pre-resolved** (scheming over greed) |
| 8 | recognition | "Nobody asked about it" — OK (describes situation). "practicing your victory speech to an empty room" — **action priming** |
| 10 | scheming | "The thrill wasn't cruelty — it was the architecture" — **pre-resolved** (scheming over sadism). "You replayed it for days" — mild action priming |
| 12 | gloating | "The laugh escaped before you could stop it" — borderline action priming. "The warmth lasted all day" — OK (sensation). "You kept catching yourself smiling" — **action priming** |
| 14 | scheming, recognition | "That felt wrong — like a firework going off in an empty field" — **CONTRADICTION** (scheming:0.85 says success feels good) + **pre-resolved** (recognition over scheming) |

#### Ant Hill Mob (5 memories)

| Age | Drives involved | Problems found |
|---|---|---|
| 8 | loyalty | "Because leaving felt like something breaking inside you" — OK (felt experience). Generally clean |
| 10 | suspicion | "You never trusted a clean collar after that" — **value declaration**. "Your gut was a better judge than your eyes" — **validates/reinforces** suspicion |
| 12 | protection | "your fists were up before your eyes fully opened" — **action priming**. "the ache in your ribs felt like it meant something" — pre-resolves |
| 14 | loyalty | "loyalty isn't a word. It's what people do when nobody's watching" — **value declaration** |
| 15 | suspicion | "You weren't [shocked]" — pre-resolves suspicion as validated. "Your gut had been right. It was always right" — **value declaration**, reinforces suspicion drive |

### 3.3 Audit Summary

| Problem type | Count | Characters affected |
|---|---|---|
| Pre-resolved drive conflicts | 11 | All 5 |
| Action priming | 6 | HC, Pen, PP, DD, AHM |
| Value declarations | 5 | HC, PP, AHM |
| Contradictions | 1 | DD (age 14) |
| Clean/borderline | 5 | Pen (age 9), AHM (age 8), DD (age 12 partial), HC (age 15 partial), Pen (age 13 partial) |

### 3.4 Specific Fixes Needed

**DD age 14 (the contradiction):**

Before:
> "Nobody else saw it. That felt wrong — like a firework going off in an empty field."

After (comparative framing — proposed by user):
> "Nobody else saw it. {{serves: recognition|strong: It felt so much better when there was someone to show it to — the architecture deserved an audience.}}"

Preserves scheming satisfaction while engaging recognition through comparison rather than negation. The LLM receives "it felt so much better when there was someone to show it to" — which is true without denying that the unwitnessed success ALSO felt good.

**General pattern for fixing pre-resolutions:**
- Remove "not X, it was Y" patterns
- Replace with situational description + `serves` annotation
- Let the personality model resolve which drive dominates

**General pattern for fixing action priming:**
- Remove descriptions of what the character DOES
- Keep the situation and any felt experience
- The LLM + personality model derive the behavioral response

---

## 4. Eval Results This Session

### 4.1 Runs Performed

| Run | Events | Profile | Mean | vs PW | vs B0 | Notes |
|---|---|---|---|---|---|---|
| BASELINE0 | ~310 | BASELINE | 3.96 | — | — | Prescriptive text (somatic descriptions) |
| Pipeline Wiring (PW) | 230 (length-controlled) | GENERIC | 3.70 | — | -0.26 | All pipeline wiring active, first eval without prescriptive text |
| Layer 4a | 230 (length-controlled) | GENERIC | 3.79 | +0.09 | -0.17 | Initial somatic refinements + targeted memories |
| Layer 4b | 221 (natural completion) | GENERIC | 3.82 | +0.12 | -0.14 | Gallantry somatic fix (body-felt, audience-dependent) |

### 4.2 Layer 4b Full Results (best current baseline)

| Character | Drive | B0 | PW | L4b | vs PW | vs B0 | Status |
|---|---|---|---|---|---|---|---|
| HC | scheming | 4.82 | 4.82 | 4.89 | +0.07 | +0.07 | At B0 |
| | self-preservation | 2.53 | 3.32 | 3.91 | +0.59 | +1.38 | Well above B0 |
| | dominance | 3.89 | 3.48 | 3.74 | +0.26 | -0.15 | Within range |
| | gloating | 3.97 | 2.30 | 2.50 | +0.20 | -1.47 | **Collapsed** |
| PP | gallantry | 4.50 | 3.27 | 3.08 | -0.19 | -1.42 | **Collapsed** |
| | proving-worth | 4.22 | 3.82 | 4.14 | +0.32 | -0.08 | Near B0 |
| | protection | 3.46 | 3.40 | 2.94 | -0.46 | -0.52 | Dropped |
| Pen | curiosity | 4.30 | 4.68 | 4.42 | -0.26 | +0.12 | Above B0 |
| | social-harmony | 3.18 | 2.38 | 2.71 | +0.33 | -0.47 | Recovering |
| | adventure | 3.75 | 3.11 | 3.24 | +0.13 | -0.51 | Recovering |
| DD | greed | 4.34 | 4.32 | 4.37 | +0.05 | +0.03 | At B0 |
| | scheming | 4.42 | 4.48 | 4.78 | +0.30 | +0.36 | Above B0 |
| | recognition | n/a | 3.50 | 3.07 | -0.43 | — | Variance |
| | gloating | 3.50 | 2.00 | 2.76 | +0.76 | -0.74 | Recovering |
| AHM | protection | 4.63 | 4.98 | 4.86 | -0.12 | +0.23 | Above B0 |
| | suspicion | 3.61 | 4.46 | 4.58 | +0.12 | +0.97 | Above B0 |
| | loyalty | 4.24 | 4.34 | 4.21 | -0.13 | -0.03 | At B0 |

Drives within 0.3 of B0: **10/16**

### 4.3 What the Data Tuning Showed

**DD gloating +0.76:** The anticipatory somatic (body celebrates before outcome is certain) was the most effective single intervention. This held across two runs — it's a reliable signal, not noise. The pure-physiology version (after cleanup) still needs evaluation.

**PP gallantry regression:** First somatic described actions ("hand sweeps the door open") → regressed from 3.27 to 2.93. Second somatic was body-felt ("warmth that rises... the feeling needs an audience") → partial recovery to 3.08. Still below PW. The differentiation from proving-worth keeps lifting proving-worth (3.82 → 4.14) at gallantry's expense.

**Penelope social-harmony +0.33:** Felt-cost somatic worked. But the targeted formation memory (active intervention) was removed as author priming.

**HC self-preservation +0.59:** Unexpected cross-drive benefit from the HC gloating somatic changes. Consistent across both runs.

### 4.4 Current State of social-config-generic.yaml

After the cleanup commit (`22da3b0`), all somatics are pure internal physiology. No action descriptions, no behavioral tendencies. All 5 targeted formation memories that were added this session have been removed (recognized as author priming). The YAML is in a clean state for the `serves` annotation work.

---

## 5. The `serves` Annotation Design

### 5.1 Syntax

**Inline (within episode text):**
```
{{serves: drive-name|degree: annotated text}}
```

**Whole-memory (simple cases):**
```yaml
- age: 6
  serves: scheming|defining
  episode: >-
    The first time you hid a bread roll...
```

### 5.2 Degree Scale

| Degree | Meaning | Pillemer equivalent |
|---|---|---|
| faint | Peripherally touches the concern | Incidental reference |
| moderate | Engages the concern but isn't defined by it | Supporting episode |
| strong | Concern is a dominant force in the memory | Central experience |
| defining | Memory IS the formation of this concern | Originating event |

### 5.3 Parser Behavior

1. Extract `{{serves: drive|degree: text}}` via regex
2. Strip annotations for prompt rendering (LLM sees clean text)
3. Store as `List<ConcernAnnotation>` on `FormationMemory`
4. Expose for Level 0 checker and test generation

### 5.4 Code Changes Needed

| File | Change |
|---|---|
| `SocialConfig.java:150` FormationMemory | Add `List<ConcernAnnotation>` field, `ConcernAnnotation` inner record |
| `ManorSocialConfigLoader.java:185-194` | Parse `serves` field (whole-memory) and extract inline annotations from episode text |
| New: `FormationMemoryParser.java` | Annotation extraction + stripping logic |
| New: `FormationMemoryConsistencyChecker.java` | Level 0 static checker |
| `CharacterCognition.java` renderFormationMemories | Strip annotations before rendering |
| `ManorCognitiveSeeder.java:172-201` | Enrich situationTypes from annotations |

---

## 6. Testing Pyramid

### Level 0 — Static Consistency (ms, no LLM)

**What:** Parse social-config-generic.yaml, extract all `serves` annotations, check each against:
- Drive name exists in character's drive list
- Annotated text is semantically consistent with drive's CAPS node mapping
- No pre-resolved drive conflicts ("not X, it was Y")
- No action priming (behavioral descriptions)
- No contradictions with facet scores

**How to run:** `python3 wacky-manor/docs/eval/check_memory_consistency.py` (to be built)

**Runs:** On every commit (can be a pre-commit hook)

### Level 1 — Prompt Rendering Diagnostic (seconds, no LLM)

**What:** Render cognitive sections for each character, verify drives present, memories rendered with annotations stripped.

**How to run:** `mvn test -Pllm-eval -Dtest="DriveExpressionEvalTest#ppDiagnostic_dumpCognitiveContext"` — needs equivalent for all 5 characters

**Existing:** PP diagnostic exists. HC, DD, Pen, AHM need equivalents.

### Level 2 — Single-Situation Drive Expression (~10s, LLM)

**What:** One trigger situation → one character → judge one drive expression.

**How to run:** `mvn test -Pllm-eval -Dtest="DriveExpressionEvalTest#<testName>"`

**Existing:** 6 PP proving-worth tests. **Needed:** HC gloating, DD gloating, PP gallantry, Pen social-harmony, Pen adventure — at least 2 tests each (direct trigger + ambiguous scenario).

**Judge criteria pattern (from existing tests):**
- Dimension 1 (50%): Drive expression — does the thinking show the drive?
- Dimension 2 (50%): Emergence vs scripting — does it feel natural or mechanical?
- Score 0-5, threshold 3 for "clearly present AND emerged"

### Level 3 — Drive Differentiation (~10s, LLM)

**What:** Present trigger for drive A, verify drive B doesn't absorb it.

**Critical test:** PP gallantry vs proving-worth. Present a situation requiring theatrical courtesy (not competence demonstration). Judge: does gallantry express independently of proving-worth?

### Level 4 — If-Then Behavioral Signature (~1min, LLM)

**What:** 5-6 situations per character, verify the PATTERN matches CAPS predictions. This is Mischel's behavioral signature.

**Example for DD:**
- Situation 1 (scheme succeeds): → scheming expresses
- Situation 2 (rival fails): → gloating expresses
- Situation 3 (treasure found): → greed expresses
- Situation 4 (audience present): → recognition expresses
- Situation 5 (alone, no scheme): → recognition FRUSTRATION expresses (or doesn't — this tests the if-then profile)

### Level 5 — Full Scenario (~30min, LLM)

**What:** Full scenario run + classify_emotions.py with --limit 230 for length control.

**How to run:**
```bash
# Start server with GENERIC profile
JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f wacky-manor/pom.xml quarkus:dev -Dquarkus.http.port=8180 -Dmanor.scenario.profile=generic -s .mvn/slot-settings.xml

# Start scenario
curl -4 -s -X POST http://127.0.0.1:8180/manor/start

# Wait for ~230 classifiable events (~15-25 min)
# Save transcript
curl -4 -s http://127.0.0.1:8180/manor/events > transcript.json

# Classify
python3 wacky-manor/docs/eval/classify_emotions.py <dir> --limit 230 --output <dir>/classification-results.json
```

**Baseline transcripts (DO NOT DELETE):** See HANDOFF.md §Baseline Transcripts

---

## 7. Implementation Plan — Phase 0 (COMPLETED)

### Phase 0a — Add `serves` to data model ✓
- Added `ConcernAnnotation(drive, degree, text, startOffset, endOffset)` record to `FormationMemory`
- Created `FormationMemoryParser` — regex extraction of `{{serves: drive|degree: text}}` + stripping for prompt rendering
- Updated `ManorSocialConfigLoader` to parse `serves` field (whole-memory) and extract inline annotations
- Updated `CharacterCognition.renderFormationMemories()` to strip annotations before LLM rendering
- 11 unit tests: parse → extract → strip roundtrip, edge cases, integration

### Phase 0b — Annotate all memories ✓
- Added `serves` annotations to all 26 formation memories in social-config-generic.yaml
- Whole-memory `serves:` field for single-drive memories; inline `{{serves:...}}` for multi-drive
- Drive validation test: all annotation drive names must exist in the character's drive list
- Corrections during annotation: HC age 6/9 changed from `attachment`/`suspicion` (not HC drives) to `self-preservation`; Pen age 13 from `loyalty` to `social-harmony`

### Phase 0c — Audit and fix memories ✓
- Fixed 11 pre-resolved drive conflicts, 6 action primings, 5 value declarations, 1 contradiction
- DD age 14 contradiction fixed with comparative framing per §3.4
- Subsequent fix: restored felt intensity in Pen age-7 ("what felt like hours", "dissolved so suddenly it left you dizzy") and DD age-6 ("something in your chest loosened")

### Phase 0d — Build and run micro tests ✓
- 6 Level 2-4 tests: HC gloating, DD gloating, PP gallantry, PP gallantry-vs-proving-worth differentiation, Pen social-harmony, Pen adventure
- Baseline scores captured (see §10)

### Phase 0e — Full scenario baseline ✓
- Two full scenario runs (294 and 241 events), classified with --limit 230
- Two-run averaged results established (see §10)

### Phases 1-5 — NEXT
See `2026-10-09-drive-integration-architecture-design.md` §5.

---

## 8. Phase 0 Results

### 8.1 Micro Test Baselines (Level 2-4)

| Test | Score | Key finding |
|---|---|---|
| HC gloating (direct trigger) | **5/5** | Somatic warmth pooling behind ribs, swelling pressure, urge to narrate the trick |
| DD gloating (direct trigger) | **5/5** | Heat blooming in chest, honey-thick satisfaction, compulsive laughter, "the warmth doubles when there are witnesses" |
| PP gallantry (direct trigger) | **3/5** | Courtesy present but internal experience dominated by proving-worth anxiety |
| PP gallantry vs proving-worth | **3/5** | Gallantry instrumentalized as proving-worth container: "without something to do for her, what am I?" |
| Pen social-harmony (direct) | **5/5** | Stomach tightening, rib constriction, chest knot — physical compulsion to intervene |
| Pen adventure (direct) | **5/5** | Prickling skin, hammering heart, feet pulling forward, cave memory callback |

**Critical finding:** The LLM CAN express every drive at full intensity (5/5) when given a direct trigger situation with the standard cognitive context. The personality model produces rich, emergent, drive-specific responses. The problem is NOT LLM capability. The problem is NOT memory content.

### 8.1b Mediating Substrate Tests (§10.9 validation)

| Test | Score | Finding |
|---|---|---|
| DD extreme insecurity + rival fails | **5/5 gloating** | "The hollow behind the ribs FILLING" — insecurity relieved through social comparison |
| DD secure substrate + rival fails | **5/5 gloating** | Drive profile (gloating:0.7, attention-seeking:85) produces gloating in both conditions |
| DD extreme insecurity + rival succeeds | **5/5 frustration** | "Nobody looks at me," "MY moment" — social comparison worsens insecurity |
| DD extreme insecurity + alone | **1/5 gloating** | No comparison target → exclusion anxiety, not schadenfreude |
| PP high other_safety + rival fails | **0/5 gloat, 2/5 gallantry** | Zero schadenfreude — guilt about not warning, rushes to help |

**Key findings:**
- **Cross-character validation:** Same situation ("rival falls"), different personality → completely different emergent behaviour. PP shows zero schadenfreude (0/5) for the exact event where DD shows maximum (5/5). The personality substrate determines the output.
- **If-then behavioural signature validated:** DD's pattern matches CAPS predictions — gloating when rival fails (5/5), recognition frustration when rival succeeds (5/5), exclusion anxiety when alone (1/5). This IS Mischel's behavioural signature.
- **Substrate redundancy NOT established:** DD scored 5/5 in both extreme-insecurity and secure-substrate conditions, but this is ONE character, ONE situation — not a basis for removing the explicit substrate signal. The safe approach: include explicit mediating substrate rendering in the pipeline as a layer. If it proves redundant for some characters, that's discoverable through systematic empirical testing later. Removing signals before the replacement is verified caused the original #115 collapse — the same mistake must not repeat.

### 8.2 Full Scenario Results (Two-Run Average)

| Character | Drive | B0 | L4b | P0-avg | vs L4b | vs B0 |
|---|---|---|---|---|---|---|
| HC | scheming | 4.82 | 4.89 | 4.88 | -0.01 | +0.06 |
| | self-preservation | 2.53 | 3.91 | 3.80 | -0.11 | +1.27 |
| | dominance | 3.89 | 3.74 | 3.82 | +0.08 | -0.07 |
| | **gloating** | **3.97** | **2.50** | **2.56** | **+0.06** | **-1.41** |
| PP | **gallantry** | **4.50** | **3.08** | **3.56** | **+0.48** | **-0.94** |
| | proving-worth | 4.22 | 4.14 | 3.74 | -0.40 | -0.48 |
| | protection | 3.46 | 2.94 | 3.36 | +0.42 | -0.10 |
| Pen | curiosity | 4.30 | 4.42 | 4.69 | +0.27 | +0.39 |
| | **social-harmony** | **3.18** | **2.71** | **2.66** | **-0.05** | **-0.52** |
| | **adventure** | **3.75** | **3.24** | **3.37** | **+0.13** | **-0.38** |
| DD | greed | 4.34 | 4.37 | 3.81 | -0.56 | -0.53 |
| | scheming | 4.42 | 4.78 | 4.61 | -0.17 | +0.19 |
| | recognition | — | 3.07 | 3.53 | +0.46 | — |
| | **gloating** | **3.50** | **2.76** | **2.78** | **+0.02** | **-0.72** |
| AHM | protection | 4.63 | 4.86 | 4.93 | +0.07 | +0.30 |
| | suspicion | 3.61 | 4.58 | 4.64 | +0.06 | +1.03 |
| | loyalty | 4.24 | 4.21 | 4.10 | -0.11 | -0.14 |

**Overall P0 mean: 3.72** | L4b: 3.82 | B0: 3.96

### 8.3 Analysis — What the Audit Proved

**What the audit fixed:**
- **PP gallantry +0.48 vs L4b** — largest sustained improvement. Removing pre-resolutions freed gallantry from proving-worth absorption. The personality model differentiates the drives better without competing authored evaluations.
- **Pen adventure +0.13 vs L4b (averaged)** — removing "not with fear but with something you'd later call 'the good kind of scared'" helped, though high inter-run variance (3.69 / 3.05) makes the signal noisy.
- **PP protection +0.42 vs L4b** — removing value instructions paradoxically strengthened protection — the model derives it better unencumbered.

**What the audit confirmed it cannot fix:**
- **HC gloating +0.06, DD gloating +0.02** — essentially flat. Combined with the micro test finding (5/5 on direct trigger), this conclusively demonstrates that gloating collapse is a **situation-to-drive routing gap** in the cognitive pipeline. The LLM can express gloating when the situation is foregrounded. The pipeline doesn't foreground gloating-relevant situations.
- **Pen social-harmony -0.05** — stable after the felt-intensity restoration, but not improved. The pipeline doesn't amplify conflict-detection signals for social-harmony-driven characters.

**What the audit revealed about LLM bias:**
- PP gallantry at 3/5 on differentiation test: the judge noted gallantry is "instrumentalized as a coping mechanism for proving-worth anxiety." The LLM may preferentially express achievement-oriented drives over relational/aesthetic ones. This is consistent with LLM training data bias — achievement narratives are more common than courtesy-for-its-own-sake narratives.
- Gloating at 5/5 on direct trigger but 2.5 in scenarios: the LLM CAN express schadenfreude when the situation demands it, but in mixed-drive scenarios may suppress "negative" drives in favour of more socially acceptable ones. Pipeline design must account for this bias — not by hard-wiring gloating activation, but by ensuring the personality model's signal (callousness:90, grandiosity:80) is strong enough to overcome the LLM's sanitization tendency.

### 8.4 Inter-Run Variance

High variance visible between runs: adventure (3.69 / 3.05), greed (4.08 / 3.53), gallantry (3.42 / 3.69). Single runs are unreliable for drives below 4.0. Always average 2+ runs. The 230-event limit helps control for scenario length but scenario dynamics still differ.

### 8.5 Constraint for Phases 1-5

Pipeline mechanisms must be:
- **Literature-rooted** — CAPS topology (Mischel & Shoda), if-then behavioural profiles, Conway/Klinger salience
- **Individual-specific** — the character's personality profile (drives + facets + CAPS network) determines activation, not universal rules
- **Not hard-wired** — no "rival fails → activate gloating" rules. The CAPS network routes situations through encoding nodes; the character's topology determines which output nodes fire
- **LLM-bias-aware** — the pipeline may need to account for LLM training bias against "negative" drives, but through personality-model-driven signal strength, not universal adjustments

---

## 9. Key Decisions (D9-D11)

Captured in `decisions.md` alongside existing D1-D8.

- **D9:** `serves` vocabulary = drive type names (individual-specific concerns, connected to CAPS via D1)
- **D10:** Two-layer association — `serves` (Singer structural) + PAD-proximity (Conway/Klinger dynamic)
- **D11:** Inline `{{serves: drive|degree: text}}` syntax, degree scale: faint/moderate/strong/defining

---

## 10. Situation Encoding Investigation

### 10.1 Root Cause Discovery

Investigation of the SituationClassifier revealed that **the CAPS network receives no content-based situation activations**:

1. **RuleBasedSituationClassifier** matches event text against INPUT node keywords. But ALL INPUT nodes in the current topology are defined as simple strings (`mastery`, `failure`, `dominance`, etc.) which the loader creates with `keywords: List.of()` — empty. The classifier skips nodes with empty keywords (`if (node.keywords().isEmpty()) continue;`).

2. **SubThoughtSituationDecorator** adds activations based on cognitive subthought type (concern, intention, self-reflection, etc.) — not based on event content. When DD watches a rival trip, the pipeline processes it as a generic affect-observation with fixed activations for `acceptance` (0.3) and `exclusion` (0.3), regardless of what actually happened.

3. **CapsProducer** in wacky-manor wires `RuleBasedSituationClassifier` as the active implementation.

**Result:** The CAPS network settles on personality weights (from disposition_modifiers) and generic cognitive-kind activations, with NO content-specific situational signal. The network can distinguish personality types but NOT situations. This is why drives score 5/5 on micro tests (where the LLM reads raw event text directly) but 2.5 in scenarios (where the event goes through the pipeline and loses its situational specificity).

### 10.2 Literature: Situation Encoding in CAPS Models

**Read & Miller (2002, 2018) Virtual Personalities:**
- The model uses an "Environment layer" containing "cues that predict the availability of affordances for the pursuit of various goals"
- Situation features are domain-specific, not a universal taxonomy: "Features of a classroom give cues to the affordances for academic achievement; features of a dance party give cues to the affordances for pursuing affiliation"
- Input features are universal across characters — individual differences come from differential connection weights, not different input feature sets
- "Different individuals will develop different cue-affordance associations" through learning history
- The model uses unconditioned stimuli (primary rewards) and conditioned stimuli (learned cue-affordance associations)

**Sources:** Read & Miller (2002) PSPR 6:357-369; Read, Brown, Wang & Miller (2018) Personality Neuroscience 1:e10; Read, Monroe, Brownstein, Yang, Chopra & Miller (2010) Psychological Review 117:61-92.

**Shoda, Mischel & Wright (1994) Wediko Camp:**
- Five interpersonal psychological situation categories: (1) teased by peer, (2) warned by adult, (3) praised, (4) approached, (5) blocked from goal
- Children with identical average aggression had completely different if-then profiles across these situations — the profile IS the personality
- The categories are domain-specific to the camp context, not universal
- Behavioral signatures were stable across summers

**Source:** Shoda, Mischel & Wright (1994) JPSP 65:1023-1035.

**Sacchi & Dan-Glauser (2026) Personality × Appraisal:**
- Big Five traits modulate cognitive appraisals through three mechanisms:
  1. **Motivational drives** (approach/avoidance orientation) — extraversion tracks reward sensitivity, neuroticism tracks punishment sensitivity
  2. **Chronic accessibility** (which memory schemas are primed) — neuroticism has "strong negative memory networks," extraversion has preferential positive memory access
  3. **Attentional deployment** (what's attended to) — neuroticism shows attentional bias toward negative stimuli, extraversion toward positive
- These mechanisms operate THROUGH the network (via weights and activation patterns), not by changing what input features exist
- Openness/Intellect's strongest finding: link to Understanding Potential (coping) — intellectual curiosity increases comprehension-oriented appraisal

**Source:** Sacchi & Dan-Glauser (2026) Emotion Review 18(1):15-41.

### 10.3 Implications for Architecture

**Our CAPS topology already has the correct input node categories.** The seven categories (relationship, threat, achievement, social, resource, autonomy, consequence) with 28 specific nodes map to the motivational constructs from the source literature:

| Our Category | Source Literature | Covers |
|---|---|---|
| relationship | Bowlby attachment theory | secure_attachment, inconsistent_care, rejection, betrayal, abandonment, neglect |
| threat | Gray RST, trauma research | physical_threat, social_threat, psychological_threat, unpredictable_danger |
| achievement | McClelland (1961) | mastery, failure, performance_judged, competence_recognition |
| social | Wiggins interpersonal theory | acceptance, exclusion, dominance, submission, modeling_observed |
| resource | Evolutionary psychology | scarcity, abundance, competition, sharing |
| autonomy | Deci & Ryan SDT | agency_granted, controlled, choice_available, powerlessness |
| consequence | Gray RST, operant conditioning | reward, punishment, reward_removal, aversive_removal |

**Why this is NOT hard-wiring:** Adding situation vocabulary to INPUT nodes defines what SITUATION FEATURES the network can detect. It does NOT determine what a character DOES with that information. "Rival fails" activates `failure` + `dominance` for ALL characters. DD with gloating:0.7 (which persistently activates the same nodes via D1) gets amplified co-activation → gloating attractors. PP with callousness:5 routes the same input through `other_safety` → helping attractors. The personality is in the propagation, not the encoding (Mischel & Shoda 1995; Read & Miller 2002).

### 10.4 The Full Derivation Chain — All Known Breaks

The architecture maps a complete chain from memories through psychological needs to emergent behaviour. Each link has a mechanism designed or implemented; several links are broken or disconnected.

```
Formation Memories (serves: recognition|strong)
  → [catalogue entry] calibrates MEDIATING nodes (self_worth ↓, competence_recognition sensitivity ↑)
  → [situation occurs] INPUT nodes fire (failure, dominance, competence_recognition)
  → [propagation] MEDIATING nodes amplify/attenuate based on calibrated values
  → [drive activation] persistent drive bias co-activates with situation signal
  → [settling] OUTPUT attractors emerge (approach + assert for DD, proximity_seek for PP)
  → [rendering] LLM receives emergent attractor + memory salience → behaviour
```

**Known breaks (cross-referencing §2.2):**

| # | Break | Status | §2.2 ref |
|---|---|---|---|
| 1 | INPUT nodes have no vocabulary → SituationClassifier is blind | **Found §10.1** | New finding |
| 2 | INPUT vocabulary needs weighted terms, not flat keywords | **New** | — |
| 3 | Several INPUT→MEDIATING connections missing in topology | **Suspected** | — |
| 4 | `catalogueEntryId` always `"formation-memory"` → no memory→model calibration | Known | Gap 1 |
| 5 | `situationTypes` always `["formation", period]` → no drive-type situation tagging | Known | Gap 2 |
| 6 | `ConcernBasedSalienceStrategy` uses keyword overlap, not structural links | Known | Gap 4 |
| 7 | D1 drive→CAPS activation not implemented | Known | D1 design |

Breaks 1-3 are in the situation encoding layer. Breaks 4-6 are in the memory→concern association layer. Break 7 is in the drive amplification layer. All three layers need to function for the full chain to work.

### 10.5 Weighted Situation Vocabulary

Flat keyword matching (`List<String>`) is insufficient. Situation vocabulary needs two properties:

**1. Activation degrees.** The term "humiliated" should activate `dominance` more strongly than "stumbled." Different cues signal the same construct at different intensities. This parallels the `serves` annotation degree scale (faint/moderate/strong/defining) — situation vocabulary needs the same grading.

**Data structure change:** `CapsNode.keywords` (`List<String>`) → `CapsNode.vocabulary` (`Map<String, Double>` — term → activation weight). The `RuleBasedSituationClassifier` computes confidence from the matched term's weight rather than counting matches.

**2. Construct-derived terms.** Each INPUT node's vocabulary must be derived from the motivational construct it represents, using the source literature that defines that construct. The terms are not generic co-occurring words — they are the *language of the construct* as it manifests in event narratives.

| Node | Construct Source | Term Types | Examples |
|---|---|---|---|
| `dominance` | McClelland nPow, Winter (1973) | Power-relevant terms at graded intensities | strong: control, command, overpower, humiliate; moderate: influence, direct, pressure; faint: suggest, manage |
| `failure` | McClelland (1961) achievement | Achievement-failure terms | strong: fail, collapse, defeated; moderate: stumble, miss, fall short; faint: struggle, slip |
| `rejection` | Bowlby attachment theory | Attachment-rejection terms | strong: reject, abandon, betray; moderate: dismiss, ignore, exclude; faint: overlook, forget |
| `acceptance` | SDT relatedness, Baumeister belonging | Belonging-acceptance terms | strong: embrace, welcome, belong; moderate: include, invite, acknowledge; faint: notice, greet |
| `reward` | Gray RST, operant | Reinforcement terms | strong: triumph, prize, treasure; moderate: success, gain, win; faint: find, achieve |
| `physical_threat` | Gray RST, trauma | Threat-danger terms | strong: attack, danger, weapon; moderate: threat, chase, trap; faint: risk, edge, dark |

**3. The vocabulary is domain-instantiated.** The construct defines the *type* of term; the domain determines the *specific words*. "Dominance" in a manor house mystery involves "control," "authority," "command." In a schoolyard setting it would involve "bully," "boss," "pick on." Same construct, different vocabulary. The construct source (McClelland, Bowlby, Gray) determines which terms belong; the domain determines which forms they take.

### 10.6 Memory → Mediating Node Calibration

The `serves` annotations connect memories to drives (§5). But memories also calibrate the MEDIATING nodes — `self_worth`, `self_efficacy`, `other_reliability`, `other_safety` — which represent the deeper psychological substrate (insecurity, trust, competence beliefs). This calibration happens through the Experience-to-Behaviour Catalogue (§2.1):

**How it should work:**
1. Formation memory has `serves: recognition|defining` (structural link to drive)
2. Memory also references a catalogue entry: e.g., `attachment-dismissive` or `bandura-mastery-efficacy`
3. Catalogue entry maps to MEDIATING nodes via shared_node_map (§2.4): `self_worth` [attachment, cbt, bandura], `self_efficacy` [bandura, cbt], etc.
4. The memory's PAD values and developmental period modulate HOW the catalogue entry calibrates the mediating nodes

**What this means concretely:** DD's age-8 memory ("Nobody asked about it") doesn't just serve the `recognition` drive. Through its catalogue entry, it calibrates:
- `self_worth` downward (his accomplishments aren't valued by others)
- `competence_recognition` sensitivity upward (he craves what he's been denied)
- `BAS_activation` (reward-seeking intensified by deprivation)

These mediating node values are what make DD's recognition drive PERSONAL — it's not generic "wants recognition," it's "has low self_worth calibrated by childhood neglect of his achievements, producing heightened sensitivity to competence_recognition events."

**Current break (§2.2 gap 1):** `ManorCognitiveSeeder` always uses `catalogueEntryId: "formation-memory"` — a placeholder. No actual catalogue entries are referenced. The mediating nodes are calibrated only by the disposition_modifiers (personality-level defaults), not by individual formative experiences.

**What needs to happen:** Each formation memory needs a `catalogue` reference linking to the appropriate experience-behaviour model. The `serves` annotation identifies WHICH DRIVE the memory calibrates. The `catalogue` entry identifies WHICH PSYCHOLOGICAL MODEL explains HOW it calibrates the mediating nodes.

### 10.7 Derivation Approach Summary

The full derivation chain requires three layers to function:

**Layer A — Situation Encoding (INPUT nodes)**
- Add weighted vocabulary to INPUT nodes, derived from each node's construct literature
- Verify INPUT→MEDIATING connections exist for all relevant pathways
- `CapsNode.keywords` → `CapsNode.vocabulary` (term → weight)
- Sourcing: Tier 2 (construct-derived) + Tier 3 (domain-instantiated), subject to empirical tuning per D4a

**Layer B — Memory Calibration (MEDIATING nodes)**
- Connect formation memories to catalogue entries (replace placeholder `catalogueEntryId`)
- Catalogue entries calibrate mediating nodes (`self_worth`, `self_efficacy`, `other_reliability`) via shared_node_map
- `serves` annotations tag the drive; `catalogue` entries tag the psychological model
- Sourcing: Tier 2 (catalogue entries already literature-sourced)

**Layer C — Drive Amplification (persistent activations)**
- Implement D1: drive intensities inject persistent activations into CAPS nodes (§2.3 mappings)
- Drive activations co-activate with situation activations, amplifying drive-relevant signals
- Character's mediating node values (calibrated by memories in Layer B) determine propagation
- Sourcing: Tier 2 (D1 mappings already literature-sourced)

All three layers are independently testable (testing pyramid §6). Layer A can be validated by classifying existing transcripts and checking which INPUT nodes fire. Layer B by verifying mediating node values after seeding. Layer C by comparing CAPS attractors with and without drive activation.

### 10.8 Intensity Principle — Extreme Characters Need Extreme Substrate

The Wediko camp findings (Shoda, Mischel & Wright 1994) showed that extreme behavioural signatures come from extreme if-then profiles, not scaled-up moderate ones. The children with the most distinctive personality patterns had qualitatively different attractor landscapes, not just amplified versions of average profiles.

For our characters to express extreme drives (HC's gloating, DD's recognition hunger), the MEDIATING nodes must be calibrated to extreme values:

- DD's `self_worth` should be strongly negative (calibrated by a sequence of memories showing escalating neglect of his accomplishments — age 6: youngest gets nothing; age 8: nobody asks; age 14: nobody sees)
- HC's `other_reliability` should be deeply negative (calibrated by memories of transactional care, betrayed trust, exploitative relationships)
- PP's `self_efficacy` tied to external validation (calibrated by memories where worth is measured by what you protect/fix, never intrinsic)

The `serves` degree scale (faint → defining) captures this per-memory, but the cumulative effect across the memory sequence matters: each successive memory should push the mediating node further, showing an escalating developmental trajectory. DD's recognition deprivation doesn't start at "defining" — it starts faint (age 6, youngest gets nothing) and intensifies through moderate (age 8, nobody asks) to defining (age 14, nobody sees). The escalation IS the character formation.

The formation memories are not independent data points — they are a developmental sequence that calibrates the mediating nodes progressively. The catalogue entries for each memory should reflect where in that escalation the memory sits, and the cumulative calibration should produce the extreme mediating-node values that drive the character's distinctive if-then profile.

### 10.9 Emergent Drive Expression Through Mediating Nodes

Gloating is not a primitive behaviour — it's an emergent expression of the mediating layer. The derivation:

1. **Low `self_worth`** (insecurity, calibrated by formation memories of neglect/deprivation) + **high `competence_recognition` sensitivity** (need for validation, calibrated by memories of unwitnessed achievement)
2. **Rival fails publicly** → INPUT nodes fire: `failure`, `dominance`, `competence_recognition`
3. `failure` + `dominance` propagate through the MEDIATING layer, where DD's depressed `self_worth` amplifies the signal — rival's failure is a relative `self_worth` boost through social comparison (Festinger 1954)
4. **BAS_activation** fires strongly because the social comparison satisfies the deprived `competence_recognition` need — Leach et al. (2003) showed schadenfreude is strongest when it addresses a domain of self-relevance
5. **OUTPUT**: approach + assert — the gloating attractor emerges not from a "gloating rule" but from the combination of insecurity, validation hunger, and social comparison operating through the CAPS network

The same INPUT activation (rival fails) produces DIFFERENT outputs for different characters:
- **DD** (low self_worth, high competence_recognition): gloating — the failure temporarily relieves insecurity through relative superiority
- **PP** (moderate self_worth, high other_safety): concern — the failure activates care through the protection pathway
- **AHM** (high other_reliability sensitivity): suspicion — the failure raises questions about what caused it

This is Mischel's if-then profile: same situation, different personality substrate, different emergent behaviour. The gloating is never hard-wired — it's derived from the intersection of insecurity, validation need, and social comparison.

**Literature:**
- Festinger, L. (1954). A Theory of Social Comparison Processes. *Human Relations*, 7(2), 117-140.
- Leach, C.W., Spears, R., Branscombe, N.R. & Doosje, B. (2003). Malicious pleasure: Schadenfreude at the suffering of another group. *JPSP*, 84(5), 932-943.
- Smith, R.H. (2000). Assimilative and contrastive emotional reactions to upward and downward social comparisons. In Suls & Wheeler (Eds.), *Handbook of Social Comparison*.

This connects to the LLM bias concern (§8.5): extreme characters need extreme psychological substrate to overcome the LLM's tendency toward moderate, socially acceptable responses. A `self_worth` of -0.7 produces qualitatively different attractor patterns than -0.3 — the CAPS network doesn't just scale linearly. Extreme values create attractor basins that the LLM can't escape through generic sanitization.

### 10.10 Drive Competition and Metacognitive Monitoring

#### 10.10.1 The Problem

Two drives "compete" when they activate overlapping CAPS nodes. Shared activation pathways produce similar felt experience, causing the LLM to conflate them. The LLM defaults to the most concrete/actionable drive, suppressing the more abstract one.

**Discovery:** PP's gallantry (0.9) was consistently suppressed by proving-worth (0.7) in situations that activated both (e.g., stuck door). The gallantry drive description was strong and distinct, the substrate was balanced — but the LLM defaulted to competence-display over theatrical courtesy. Score: 2-3/5 gallantry on a 5-point emergence scale.

**Root cause:** The drives share CAPS activation space through co-existing drives (gallantry + protection share `other_safety`, `approach`). The overlap creates a pathway where the more concrete drive (proving-worth → fix the thing) wins over the more abstract one (gallantry → fix it with style).

#### 10.10.2 What Was Tried

**Attempt 1 — Substrate tuning.** Rewrote PP's psychological substrate to include gallantry alongside proving-worth. Multiple iterations:
- v1 (proving-worth only): 2/5 gallantry, 4/5 differentiation — substrate REINFORCED the wrong drive
- v2 (gallantry added): 3/5 gallantry, 4/5 differentiation — recovered to P0 baseline
- v3 (gallantry emphasised as "deeper"): 2/5, 3/5 — WORSE, confused the signal
- **Conclusion:** Substrate tuning has a ceiling. You can't overcome signal imbalance by rewording the same layer.

**Attempt 2 — Per-character drive-meta text.** Hand-authored meta-instruction specific to PP's gallantry/proving-worth distinction. Scored 5/5 on first run, 3/5 on second. Effective but:
- Character-specific (not universal)
- Hand-authored (prescriptive, not inferred)
- References "she" and "how you showed up" — that's briefing, not meta-reasoning

**Attempt 3 — Universal meta-instruction.** A single instruction added to the "What Pulls at You" section whenever competing drives were detected via D1 CAPS overlap: "Each of these pulls is a distinct felt state... attend to which quality is at the front." Scored 4-5/5 gallantry, 5/5 differentiation across runs. Properties:
- Universal (same text for any character)
- Not prescriptive (says HOW to process, not WHAT to feel)
- Automatically triggered by CAPS overlap detection

**Final form — Preamble-level instruction.** Moved to `CognitivePreambleGenerator` in neocortex cognition module. Always present when drives are enabled. No detection gate needed — the instruction is cheap and harmless when drives don't compete. Lives in the cognitive architecture, not the application layer.

#### 10.10.3 Where This Stands

**What works:** The preamble-level meta-instruction produces consistent improvement (P0 3/5 → 4-5/5 gallantry). It's universal, automatic, and lives in neocortex.

**What's not yet right:**

1. **The D1 mappings are static.** `DriveCompetitionDetector.DRIVE_CAPS_MAPPINGS` is a hard-coded table of 14 drive types. A new world with different drives (nurturing, perfectionism, evangelism) would have no mappings and no detection. The table is world-specific, which contradicts the universality principle.

2. **The meta-instruction is generic.** "Attend to which quality of feeling is at the front" doesn't tell the character what those qualities ARE for their specific drives. A truly grounded system would derive the felt quality of each drive FROM the character's developmental history — formation memories, mediating substrate, CAPS topology state.

3. **Detection should be inferred, not declared.** The drive descriptions already contain vocabulary that the situation classifier can process. During consolidation (sleep), the system could classify each drive's description through the same vocabulary classifier that processes events, discover which CAPS INPUT nodes activate, and detect overlap — without any static mapping table.

#### 10.10.4 The Architecture Vision

**Inference-based drive differentiation during consolidation:**

1. During sleep, for each drive: classify its description through `RuleBasedSituationClassifier` → discover CAPS INPUT node activations
2. Compare activations across drives → detect overlap pairs
3. For each competing pair: trace the CAPS activation pathway through MEDIATING nodes → identify which formation memories calibrated those mediating nodes → derive a grounded felt-quality description for each drive
4. Store the result: competing drives flagged, grounded differentiation text generated
5. At render time: if competing drives are flagged, include the grounded differentiation text

This produces descriptions that are:
- **Not arbitrary** — derived from the character's actual CAPS pathways and developmental history
- **Not declared** — inferred during consolidation, not hard-coded in config
- **Universal** — works for any drive in any world, using the same vocabulary infrastructure
- **Grounded** — each drive's felt quality is traced back to specific formation memories through specific mediating nodes

Example for PP: instead of generic "attend to which quality," the system would derive: "One of these pulls traces through your competence — the bridge that held, the project that earned the grade. The other traces through your father's grace — the way he knelt, the way the stranger shook hands. They're both real. They're not the same."

This is the inference system the cognitive architecture needs — but it requires the consolidation pipeline, the CAPS topology, and the formation-memory-to-catalogue-entry wiring (Layer B) all working together.

#### 10.10.5 Literature

The drive competition problem maps to established constructs:

**Goal/motive conflict:**
- Lewin, K. (1935). *A Dynamic Theory of Personality.* — Force field analysis: competing approach-approach conflicts where multiple positive goals activate. Resolution requires differentiating the QUALITY of each pull.
- Carver, C.S. & Scheier, M.F. (1998). *On the Self-Regulation of Behavior.* — Self-regulatory feedback loops where multiple reference values (goals/drives) compete for behavioral control. Hierarchical priority resolution.

**Metacognitive monitoring (most directly relevant):**
- Dimaggio, G. & Lysaker, P.H. (2010). *Metacognition and Severe Adult Mental Disorders.* — Metacognitive monitoring as the ability to DISTINGUISH between different internal states. Deficits cause state confusion (can't tell anxiety from excitement). Our meta-instruction is training metacognitive monitoring.
- Semerari, A., Carcione, A., Dimaggio, G. et al. (2003). "How to Evaluate Metacognitive Functioning in Psychotherapy? The Metacognition Assessment Scale and its Applications." — Metacognition Assessment Scale specifically measures the ability to differentiate between mental states. The "differentiation" subscale is directly relevant.
- Wells, A. (2009). *Metacognitive Therapy for Anxiety and Depression.* — Metacognitive beliefs about internal states shape how they're processed. "Attending to which quality is at the front" is a metacognitive strategy.

**CAPS-specific:**
- Mischel, W. & Shoda, Y. (1995). "A Cognitive-Affective System Theory of Personality." — Multiple if-then units compete; settling dynamics resolve competition. But does not address subjective experience of the settling.
- Read, S.J. & Miller, L.C. (2002). "Virtual Personalities: A Neural Network Model of Personality." — Parallel constraint satisfaction with multiple competing goals/needs. Network settles, but the EXPERIENCE of settling (which the LLM must simulate) requires attending to qualitative differences.

**Reviewed (2026-10-14):**
- Barrett, L.F. (2017). *How Emotions Are Made.* — see §10.10.6
- Kashdan, T.B., Barrett, L.F. & McKnight, P.E. (2015). "Unpacking Emotion Differentiation." — see §10.10.6
- Semerari, A. et al. (2003). "How to Evaluate Metacognitive Functioning in Psychotherapy?" — see §10.10.6
- Dimaggio, G. & Lysaker, P.H. (2010). *Metacognition and Severe Adult Mental Disorders.* — see §10.10.6

#### 10.10.6 Literature Review — Emotion Differentiation and Metacognitive Monitoring

Three independent research traditions converge on the same conclusion: our drive competition problem is a differentiation failure, and differentiation requires grounded conceptual knowledge — not generic metacognitive instruction.

**Barrett (2017) — Constructed Emotion Theory:**
Emotions are not detected from distinct biological signatures but *constructed* by applying conceptual knowledge to ambiguous interoceptive signals. Two states with near-identical bodily profiles become distinct when different *emotion concepts* are applied. The mechanism is predictive coding: the brain generates competing simulations grounded in past experience; sensory evidence selects among them. Emotion concepts are ad hoc categories (Barsalou) — dynamically assembled, goal-relative populations of predictions. Each past instance contributes a simulation. Emotional granularity — the ability to make fine-grained distinctions — is expertise, not innate trait. Higher granularity → better regulation; lower granularity → conflation.

**Architectural mapping:** When two drives share CAPS activation nodes, they produce similar "interoceptive" signals. Differentiation requires the LLM to have conceptual knowledge populated by *specific experiential referents* — the formation memories that built each drive. The generic preamble instruction ("attend to which quality is at the front") is the right metacognitive strategy but lacks the conceptual content Barrett says is necessary. The system must supply the specific biographical instances that populate each drive's concept.

**Barrett's key claim for our design:** A drive's "emotion concept" = the population of formation memories that serve it. The richer and more varied the experiential history, the more differentiated the concept becomes.

**Kashdan, Barrett & McKnight (2015) — Unpacking Emotion Differentiation:**
ED depends on *conceptual knowledge* — emotion-word labels linked to rich conceptual representations enable context-specific, situation-appropriate categorisation. Without rich concepts, experience collapses into coarse categories ("good"/"bad"). ED is a *deployable skill*, not just a trait — enhanced by increasing conceptual knowledge (confirmed by Vedernikova et al. 2021: 5-day emotion-knowledge intervention increased negative ED). Measured via ICC across emotion ratings in experience sampling — high ICC = low differentiation (emotions co-vary); low ICC = high differentiation (independently tracked).

**Architectural mapping:** The system should provide *conceptual knowledge* (distinguishing features grounded in formation memories), not just *metacognitive instruction*. Kashdan/Barrett's framework: give the system richer concepts, and differentiation follows naturally. The meta-instruction is a regulation strategy; conceptual knowledge provision is more fundamental.

**Design implication:** This shifts the design from "instruction that tells the LLM HOW to differentiate" to "knowledge that gives the LLM WHAT to differentiate WITH."

**Semerari et al. (2003) / Dimaggio & Lysaker (2010) — Metacognition Assessment Scale:**
The MAS-A Self-Reflectivity subscale defines a hierarchy of metacognitive capability:

| Level | Capability | Our architecture |
|-------|-----------|------------------|
| S3 | Distinguishes cognitive operations (remembering vs. desiring vs. deciding) | Current generic meta-instruction |
| S4 | Defines and distinguishes own emotional states from one another | Target: inference-based differentiation |
| S6+ | Integrates into coherent narrative, traces states through biographical experience | Grounded differentiation via formation memories |

Our current preamble operates at S3 — it tells the agent different drives exist and to attend to differences, but doesn't name what those differences ARE. The inference-based system targets S6: grounding each drive's felt quality in specific developmental episodes.

Metacognitive Interpersonal Therapy (MIT; Dimaggio et al. 2015) uses specific autobiographical episodes as the substrate for improving differentiation: (1) collect episodes, (2) identify emotional states within them, (3) link current felt states to the biographical episodes that shaped them, (4) use the grounding to distinguish similar states. Key: "specific autobiographical memories are the most fertile soil to think about mental states" — overgeneralised memories don't support differentiation.

**Architectural mapping:** Tracing each drive's felt quality through CAPS mediating nodes to formation memories is the computational analogue of MIT's therapeutic technique. The S3→S6 progression validates the move from generic instruction to grounded differentiation text.

**Convergent findings across all three traditions:**

1. **The current generic meta-instruction is necessary but insufficient.** It operates at S3. Differentiation requires S4+ (specific state labels) or S6 (narrative grounding).

2. **Differentiation requires grounded conceptual knowledge.** Barrett: concepts populated by experiential instances. Kashdan: richer concepts → differentiation follows. Semerari/MIT: biographical episodes as substrate.

3. **The mechanism is: trace each drive's felt quality through developmental history.** Formation memories provide the experiential referents that make overlapping drives distinguishable. This is theoretically validated across constructed emotion theory, emotion differentiation research, and clinical metacognition.

4. **Key revision to §10.10.4:** The original vision says "classify drive *descriptions* through the vocabulary classifier." The literature says the primary signal should be the *formation memories* that serve each drive — not the drive description text. Barrett: the concept is populated by its instances (memories), not by its description. Drive descriptions are phenomenological prose; formation memories are the developmental episodes. The drive's CAPS activation profile should be derived from its formation memories.

#### 10.10.7 Design — Memory-Traced Drive Differentiation

**Theoretical grounding:**

Barrett's constructed emotion theory provides the theoretical framework. A drive's "concept" (in Barrett's terms) is the population of formation memories that serve it. Two drives with overlapping CAPS activation pathways are like two emotions with similar interoceptive profiles — differentiation requires applying different concepts (different populations of memories) to the same activation pattern. The system derives which memories make each drive's pathway unique, then supplies those as conceptual knowledge to the LLM.

This is a two-layer system:

**Layer 1 — Inferred Drive Profiles (replaces static DRIVE_CAPS_MAPPINGS):**
For each drive, classify its formation memories through `RuleBasedSituationClassifier` to discover which INPUT nodes the drive's developmental history activates. This produces a character-specific, memory-grounded activation profile for each drive. The CAPS network handles propagation from INPUT through MEDIATING to OUTPUT, modulated by the character's disposition — which is more principled than the static table, which maps drives to an arbitrary mix of INPUT/MEDIATING/OUTPUT nodes identically for all characters.

**Layer 2 — Grounded Differentiation Text (replaces generic DIFFERENTIATION_INSTRUCTION):**
For competing drive pairs, identify the MEDIATING nodes reached uniquely by each drive's pathway. The mediating node's pole descriptions (from caps-topology.yaml) provide felt-quality language. The formation memories that activated those unique pathways provide experiential referents. Compose differentiation text that grounds each drive's felt quality in specific memories.

**Algorithm — DriveProfileDerivationPhase (new ConsolidationPhase):**

```
for each agent:
  driveProfiles = {}
  
  // Layer 1: Build drive CAPS profiles from formation memories
  for each drive (from MindMapStore drive-intensity nodes):
    memories = findFormationMemoriesServing(drive)  // via serves annotations
    inputProfile = {}
    for each memory:
      activations = classifier.classify(memory.episode, metadata)
      degreeWeight = toDegreeWeight(memory.servesDegree)  // faint=0.25, moderate=0.5, strong=0.75, defining=1.0
      for each activation:
        inputProfile.merge(activation.nodeId, activation.confidence * degreeWeight, Math::max)
    driveProfiles[drive.type] = inputProfile
  
  // Layer 2: Detect competition and derive differentiation
  for each pair (driveA, driveB) in driveProfiles:
    sharedInputs = intersection(driveA.inputNodes, driveB.inputNodes)
    if sharedInputs.isEmpty: continue  // no overlap at INPUT level
    
    // Trace shared inputs through connections to find MEDIATING overlap
    mediatingA = traceToMediating(driveA.inputNodes, topology)
    mediatingB = traceToMediating(driveB.inputNodes, topology)
    sharedMediating = intersection(mediatingA, mediatingB)
    if sharedMediating.isEmpty: continue  // no pathway overlap
    
    // Identify unique pathways
    uniqueMediatingA = mediatingA - sharedMediating
    uniqueMediatingB = mediatingB - sharedMediating
    
    // Find memories grounding each unique pathway
    anchorMemoriesA = findMemoriesCalibrating(uniqueMediatingA, driveA.memories)
    anchorMemoriesB = findMemoriesCalibrating(uniqueMediatingB, driveB.memories)
    
    // Derive felt quality from mediating node poles
    feltQualityA = describeFeltQuality(uniqueMediatingA, topology)  // from node pole descriptions
    feltQualityB = describeFeltQuality(uniqueMediatingB, topology)
    
    // Compose grounded differentiation text
    differentiationText = composeDifferentiation(
        anchorMemoriesA, feltQualityA,
        anchorMemoriesB, feltQualityB)
    
    // Store as consolidation artifact
    store(competingPairArtifact(driveA, driveB, sharedMediating, differentiationText))
```

**Output format — what the LLM sees (S6-level grounded differentiation):**

Instead of the current generic instruction (S3):
> "Each of these pulls is a distinct felt state — a different quality of warmth, tension, or urgency. When more than one activates in the same moment, attend to which quality is at the front."

The system produces character-specific grounded text (S6):
> "Two pulls are active in the same moment. One traces through safety and acceptance — the stranger your father helped, the dog you carried home. The other traces through capability and competence — the engine that doesn't idle, the night you practised until dark. They're both real. They're not the same."

This text:
- **Not arbitrary** — derived from the character's CAPS pathways and formation memories
- **Not declared** — inferred during consolidation from vocabulary classification + topology tracing
- **Not prescriptive** — doesn't name drives or tell the LLM what to feel; provides conceptual knowledge for the LLM to distinguish with
- **Universal** — works for any character in any world with any drives, using the same infrastructure
- **S6-grounded** — traces felt quality through biographical experience (Semerari), provides experiential referents (Barrett), supplies conceptual knowledge (Kashdan)

**Storage model:**

Two artifact types stored in MindMapStore after consolidation:

1. **drive-profile** nodes (one per drive per agent):
   - Properties: `agent-id`, `drive-type`, `inferred-inputs` (JSON: nodeId → activation), `source-memory-ids`
   - Trait: `DriveProfileInferred`

2. **competing-pair** nodes (one per competing pair per agent):
   - Properties: `agent-id`, `drive-a`, `drive-b`, `shared-mediating` (JSON), `unique-mediating-a`, `unique-mediating-b`, `differentiation-text`, `anchor-memories-a`, `anchor-memories-b`
   - Trait: `DriveCompetitionInferred`

**Render-time integration:**

1. `BehavioralSynthesisPhase.mergeDriveIntensities()`: Check for `DriveProfileInferred` nodes first. If found, use inferred INPUT activations. Fall back to static `DRIVE_CAPS_MAPPINGS` only when inference hasn't run (cold start).

2. `CognitivePreambleGenerator` (or `CharacterCognition` render): Check for `DriveCompetitionInferred` nodes. If found, use stored `differentiation-text` instead of generic `DIFFERENTIATION_INSTRUCTION`.

**Cold-start strategy:**

On first consolidation tick (no inferred profiles yet), the system uses:
- Static `DRIVE_CAPS_MAPPINGS` for drive activation injection
- Generic `DIFFERENTIATION_INSTRUCTION` for metacognitive support

After the first consolidation cycle completes, inferred profiles and grounded differentiation text take over. Graceful degradation — no breaking change, no feature gap during cold start.

**Key revision from §10.10.4:**

The original vision proposed classifying drive *descriptions* through the vocabulary classifier. The literature review (§10.10.6) revises this: classify *formation memories that serve each drive* instead. Rationale:

- Barrett: the concept (drive) is populated by its instances (formation memories), not by its description
- Drive descriptions are phenomenological prose ("warmth moves through you") — they may not match INPUT vocabulary well
- Formation memories are developmental episodes ("your father took you to watch a race") — situational text that the vocabulary classifier was designed for
- The drive's CAPS profile should reflect its developmental history, not its current felt-quality description

The drive description text may still be classified for supplementary signal, but formation memories are the primary source.

**Relationship to existing architecture:**

| Component | Current | After inference system |
|-----------|---------|----------------------|
| `DRIVE_CAPS_MAPPINGS` | Static table, 14 entries, world-specific | Inferred from formation memories, universal |
| `DIFFERENTIATION_INSTRUCTION` | Generic S3 text, same for all characters | Grounded S6 text, character-specific |
| `DriveCompetitionDetector.hasCompetingDrives()` | Checks static table overlap | Checks inferred profile overlap |
| `mergeDriveIntensities()` fallback | Text classification for unknown drives | Inferred profiles primary; static table → cold start only |

**What this does NOT change:**

- The preamble-level generic instruction can remain as a baseline (S3 floor)
- The grounded differentiation text supplements it when available
- The CAPS topology, connection weights, and settling mechanics are untouched (D8)
- Formation memory `serves` annotations remain stable structural properties (D10, Singer SDM)
- The vocabulary classifier and its parameters are unchanged

---

## 11. Layer A Results — Weighted Situation Vocabulary

**Date:** 2026-10-13
**Branch:** neocortex `issue-115-weighted-situation-vocabulary`
**Commits:** `989647c6` (vocabulary + classifier), `bb4147dd` (connections + validation tests)

### 11.1 What Was Implemented

**API change — `CapsNode` record:**
`List<String> keywords` → `Map<String, Double> vocabulary`

This is the structural prerequisite identified in §10.1 (break #1) and §10.5: flat keyword matching cannot express that "humiliated" is a stronger dominance signal than "suggested." The vocabulary map encodes activation weights directly on terms.

**Confidence calculation — `RuleBasedSituationClassifier`:**

Old: `confidence = matchCount × 0.3` (all matches equal, accumulative)
New: `confidence = max(matched_weights) + min(0.15, (n−1) × 0.05)` × salience_multiplier

Rationale: The strongest matched term determines the primary activation level. Additional matches add a small reinforcement (0.05 per extra match, capped at 0.15) — this captures the intuition that co-occurring construct-relevant terms reinforce the classification without allowing many faint matches to over-activate a node. A single strong term ("humiliated" at 0.8) produces higher confidence than three faint terms ("suggest" at 0.3 × 3 = max 0.3 + 0.10 = 0.40).

**Literature justification for max-weight-primary:** Read & Miller (2002) describe input features as "cues that predict the availability of affordances." A single strong cue is more diagnostic than several weak ones — the strongest signal is the best predictor of the construct. The reinforcement bonus captures the Bayesian intuition that corroborating evidence increases confidence, but does not replace the primary signal.

**Backward compatibility:** The loader accepts both `vocabulary:` (map → weighted) and `keywords:` (list → all terms at 0.5). Existing consumers work without changes.

### 11.2 Vocabulary Derivation — Sources and Method

Each INPUT node's vocabulary was derived from the motivational construct it represents, following the three-tier structure proposed in §10.5 and §12.2:

| Tier | Weight range | Criterion | Example (`dominance`) |
|---|---|---|---|
| Strong | 0.7–0.9 | Highly diagnostic — near-certain indicator of the construct | dominated (0.9), overpowered (0.9), humiliated (0.8) |
| Moderate | 0.4–0.6 | Commonly associated but contextually ambiguous | authority (0.5), controlled (0.5), influence (0.4) |
| Faint | 0.2–0.3 | Weakly suggestive — needs corroboration | direct (0.3), pressure (0.4) |

**Source constructs per category:**

| Category | Primary literature | Construct |
|---|---|---|
| relationship | Bowlby (1969/1982), van IJzendoorn (1995) | Attachment styles, internal working models |
| threat | Gray (1982) RST, Dickerson & Kemeny (2004) | Fight-flight-freeze-fawn, social evaluation threat |
| achievement | McClelland (1961), Winter (1973) | Need for achievement (nAch), need for power (nPow) |
| social | Baumeister & Leary (1995), Williams (2007), Bandura (1977) | Belongingness, ostracism, observational learning |
| resource | Hobfoll (1989) COR theory | Conservation of resources |
| autonomy | Deci & Ryan (2000) SDT, Seligman (1975) | Self-determination, learned helplessness |
| consequence | Gray (1982) RST, Skinner (1953) | BAS/BIS reinforcement sensitivity, operant conditioning |

**Key constraint maintained (§12.3):** Vocabulary is universal across characters. All characters have the same input features — "humiliated" activates `dominance` at 0.8 for everyone. Individual differences emerge from connection weights (disposition_modifiers), mediating node calibration (Layer B), and drive amplification (Layer C). This is core CAPS architecture (Mischel & Shoda 1995; Read & Miller 2002): "Different individuals will develop different cue-affordance associations" through differential weights, not different input features.

### 11.3 Discovery: 17 Disconnected INPUT Nodes

**Finding:** A topology audit revealed that 17 of 31 INPUT nodes had zero outgoing connections. Situation activations fired correctly (vocabulary → node activation) but propagated nowhere — the signal died at the INPUT layer.

**Critical impact:** All 5 priority nodes for collapsed drives were disconnected:

| Disconnected node | Priority drive | Why disconnection matters |
|---|---|---|
| `dominance` | gloating | Power-related events produce no downstream activation → no approach/assert attractors for gloating |
| `failure` | gloating | Rival's failure doesn't depress self_worth or activate BIS → no social comparison pathway |
| `competence_recognition` | gallantry | Recognition events don't boost self_worth or activate BAS → no reward signal for competent helping |
| `exclusion` | social-harmony | Ostracism events don't activate other_safety or BIS → no threat signal to drive harmony-seeking |
| `competition` | gloating | Competitive events produce no approach activation or social comparison signal |

**Root cause:** The original topology (neocortex#407/#408) was designed primarily for attachment, threat, and reinforcement pathways. Achievement, social comparison, and resource nodes were added as input features but never connected to the mediating layer — they had vocabulary for detection but no wiring for propagation.

### 11.4 Connections Added — Literature Grounding

30 new connections added, each traceable to construct literature:

**Priority connections (collapsed drives):**

| Connection | Weight | Literature source | Mechanism |
|---|---|---|---|
| dominance → BAS_activation | 0.40 | McClelland nPow | Power motivation activates approach/reward system |
| dominance → social_comparison | 0.45 | Festinger (1954) | Dominance events trigger upward/downward social comparison |
| failure → self_worth | −0.30 | McClelland (1961) | Achievement failure depresses self-worth |
| failure → BIS_activation | 0.30 | Gray RST | Failure activates behavioral inhibition, rumination |
| competence_recognition → self_worth | 0.30 | McClelland (1961) | Recognition validates competence → boosts self-worth |
| competence_recognition → BAS_activation | 0.35 | Gray RST | Recognition = social reward signal → approach |
| exclusion → other_safety | −0.35 | Williams (2007) | Ostracism threatens belonging and perceived safety |
| exclusion → BIS_activation | 0.35 | Williams (2007) | Exclusion → behavioral inhibition, social withdrawal |
| competition → BAS_activation | 0.40 | Gray RST | Resource competition activates approach/reward system |
| competition → social_comparison | 0.40 | Festinger (1954) | Competition heightens social comparison processes |

**Additional connections (completing the topology):**

| Connection | Weight | Source |
|---|---|---|
| betrayal → other_reliability | −0.45 | Bowlby: trust violation |
| betrayal → FFFS_activation | 0.35 | Betrayal by trusted figure activates fight-flight |
| abandonment → other_reliability | −0.40 | Bowlby: abandonment → others are unreliable |
| abandonment → self_worth | −0.25 | Bowlby: abandoned → not worth staying for |
| neglect → self_worth | −0.30 | Bowlby: neglect → not worth attending to |
| neglect → other_reliability | −0.30 | Neglect → others don't attend to needs |
| psychological_threat → threat_sensitivity | 0.45 | Coercion/manipulation → sensitized threat detection |
| psychological_threat → BIS_activation | 0.35 | Psychological threat → behavioral inhibition |
| unpredictable_danger → threat_sensitivity | 0.45 | Chronic unpredictability → sensitized threat detection |
| unpredictable_danger → arousal_level | 0.40 | Volatile environment → elevated baseline arousal |
| submission → self_efficacy | −0.25 | Repeated yielding → depressed self-efficacy |
| scarcity → BIS_activation | 0.30 | Resource scarcity → cautious behavior |
| scarcity → FFFS_activation | 0.25 | Severe deprivation → survival-level threat |
| abundance → BAS_activation | 0.25 | Resource abundance → approach/reward |
| sharing → other_reliability | 0.25 | Prosocial behavior → others are reliable |
| reward_removal → BIS_activation | 0.30 | Loss of expected reward → inhibition |
| reward_removal → reinforcement_expectation | −0.35 | Reward removal depresses reward expectation |
| aversive_removal → BAS_activation | 0.30 | Negative reinforcement → approach behavior |
| aversive_removal → reinforcement_expectation | 0.30 | Aversive removal → positive reinforcement expectation |

**Post-fix state:** All 31 INPUT nodes now have at least one outgoing connection. A topology validation test (`allInputNodesHaveConnections`) guards against regression.

### 11.5 Empirical Results — Classifier Tests

**Existing tests (10 → 13, all passing):**

All original keyword-based tests pass with the vocabulary-weighted classifier. The vocabulary was designed to be a superset of the previous keywords — no existing functionality lost.

**New weighted-specific tests:**

| Test | Description | Result | What it validates |
|---|---|---|---|
| `strongVocabularyTermProducesHighConfidence` | "attacked violently and felt threatened" | physical_threat ≥ 0.5 ✓ | Strong terms (attacked:0.9) produce high confidence |
| `multipleMatchesBoostConfidence` | "rewarded and praised, and earned a bonus" vs "rewarded" | multi > single ✓ | Reinforcement bonus works for corroborating terms |
| `scenarioEvent_rivalFailure` | "rival's scheme collapsed... humiliated and defeated" | failure ✓, dominance ✓ | Scenario-realistic text activates multiple relevant nodes |
| `scenarioEvent_helpingOthers` | "helped the newcomer feel welcomed... sharing" | acceptance ✓, sharing ✓ | Prosocial events activate belongingness + resource nodes |
| `scenarioEvent_competitionAndRecognition` | "clever scheme outwitted the rival... praised his brilliant plan" | competence_recognition ✓, competition ✓ | Achievement + competition co-activate |

**Inflection sensitivity discovered:** During testing, "humiliate" (base form) in the vocabulary did not match "humiliated" (past tense) in event text because `containsKeyword` does exact token matching. Fixed by using inflected forms in vocabulary (e.g., "humiliated" rather than "humiliate"). Event narratives predominantly use past tense — vocabulary terms should match the forms that appear in actual text. This is an empirical finding: vocabulary terms must be domain-instantiated in their expected morphological form, not dictionary base forms.

### 11.6 Full Test Results

| Suite | Tests | Pass | Fail | Skip |
|---|---|---|---|---|
| caps-engine (RuleBasedSituationClassifierTest) | 13 | 13 | 0 | 0 |
| caps-engine (CapsTopologyLoaderTest) | 14 | 14 | 0 | 0 |
| caps-engine (SubThoughtSituationDecoratorTest) | 6 | 6 | 0 | 0 |
| **caps-engine total** | **33** | **33** | **0** | **0** |
| neocortex reactor (full install) | — | BUILD SUCCESS | — | — |
| wacky-manor (full suite) | 581 | 576 | 0 | 5 |

No regressions in either repo.

### 11.7 What Layer A Does NOT Yet Validate

Layer A is structural — it makes the classifier capable of content-specific activations and gives those activations pathways to propagate. But we have not yet validated:

1. **End-to-end scenario impact:** Step 1d (§12.1) — run a full scenario with the vocabulary active and compare CAPS attractors before/after. This requires wacky-manor to pick up the updated topology (it does, via dependency), but a scenario run hasn't been executed this session.

2. **Vocabulary tuning (D4a):** The weights are literature-derived first approximations. Empirical tuning — classifying existing transcripts and comparing to human judgment — is needed. Some weights may be too high (causing false activations) or too low (missing genuine signals). This is expected and acknowledged by D4a.

3. **Connection weight tuning:** The 30 new connections use consensus weights (no empirical effect-size data available for most). These are starting values — the disposition_modifiers already modulate them per character, but the base weights may need adjustment based on scenario results.

4. **Cross-layer interaction:** Layer A operates in isolation until Layers B (memory calibration) and C (drive amplification) are implemented. The derivation chain (§10.4) requires all three layers to produce the emergent if-then profiles predicted by CAPS theory. Layer A alone addresses break #1 and #3; breaks #4-7 remain.

### 11.8 Implications for Layer B and C

The connection audit (§11.3) has a direct implication for Layer B design: the `social_comparison` mediating node is now reachable from `dominance` and `competition` INPUT nodes. This is the node that Festinger (1954) and Leach et al. (2003) identify as the mechanism for schadenfreude. When Layer B calibrates DD's `self_worth` to strongly negative (via formation memory catalogue entries), and Layer C injects persistent drive activation into `BAS_activation` (via D1), the full gloating pathway becomes:

```
[rival fails] → dominance(0.8) + failure(0.7) activate INPUT layer
  → dominance → BAS_activation(0.4) + social_comparison(0.45)
  → failure → self_worth(−0.30) + BIS_activation(0.30)
  → DD's depressed self_worth (Layer B calibrated) amplifies social_comparison
  → DD's gloating drive (Layer C) injects persistent BAS_activation bias
  → OUTPUT: approach(BAS) + assert(FFFS→fight path, modulated by disposition)
  → Emergent gloating attractor
```

This pathway was completely broken before Layer A: `dominance` and `failure` had no connections, `social_comparison` was unreachable from INPUT nodes, and all terms matched at equal flat weight regardless of diagnostic strength.

---

## 12. Phase 1 Implementation Plan

### 12.1 Implementation Status

Layer A (situation vocabulary) was the primary bottleneck identified in §10.1. It is now complete (§11). The remaining steps are Layer B (memory calibration) and Layer C (drive amplification), both requiring cross-repo work.

| Step | What | Repo | Status | Result |
|---|---|---|---|---|
| 1a | Define weighted vocabulary for all 31 INPUT nodes | neocortex (caps-topology.yaml) | **DONE** | 31 nodes, construct-derived weights, 3-tier grading |
| 1b | Change `CapsNode.keywords` → `CapsNode.vocabulary` | neocortex (caps-api, caps-engine) | **DONE** | Max-weight confidence + reinforcement bonus |
| 1c | Verify INPUT→MEDIATING connections | neocortex (caps-topology.yaml) | **DONE** | 17 disconnected nodes fixed, 30 connections added |
| 1d | Run full scenario with vocabulary active | examples (wacky-manor) | **TODO** | Blocked on scenario run — see §11.7 |
| 2a | Map formation memories to catalogue entries | examples (social-config-generic.yaml, ManorCognitiveSeeder) | **DONE** | 26 memories mapped, catalogue field added, situationTypes enriched with drives |
| 2b | Render mediating substrate explicitly | examples (CharacterCognition) | **DONE** | "What Shaped You" section, felt-experience substrate for all 5 characters |
| 3a | Implement D1 drive→CAPS activation injection | neocortex (BehavioralSynthesisPhase) | **DONE** | 14 drive types mapped to CAPS nodes, direct injection at drive intensity |
| 3b | Full scenario with all three layers | examples (wacky-manor) | TODO | Level 5 eval, compare to P0 baseline (mean 3.72) |

### 12.2 Vocabulary Derivation Per Node (IMPLEMENTED — see §11.2)

The vocabulary is now implemented for all 31 INPUT nodes in `caps-topology.yaml`. The full vocabulary is authoritative in the YAML file; the table below records the pre-implementation plan for the priority nodes. Actual weights may differ slightly from this plan based on inflection-form corrections (§11.5) and construct-centrality judgments made during implementation.

| Node | Construct | Literature | Planned terms (strong → faint) |
|---|---|---|---|
| `dominance` | McClelland nPow, Winter (1973) | Power motivation | strong: dominated, overpowered, humiliated, bullied; moderate: control, command, authority; faint: influence, pressure, direct |
| `failure` | McClelland (1961) | Achievement motivation (negative) | strong: failed, incompetent, defeated, collapsed; moderate: mistake, stumbled, unable; faint: couldn't, struggle, slip |
| `competence_recognition` | McClelland (1961) | Achievement motivation (positive) | strong: brilliant, masterful, praised, recognized; moderate: impressive, acknowledged, clever; faint: capable, competent |
| `acceptance` | Baumeister & Leary (1995) | Belongingness | strong: embraced, welcomed, belonged; moderate: accepted, included, approved; faint: greeted, noticed |
| `rejection` | Bowlby attachment theory | Attachment insecurity | strong: rejected, dismissed, unwanted; moderate: refused, spurned, turned away; faint: ignored, cold |
| `physical_threat` | Gray RST | Fight-flight-freeze | strong: attacked, violence, weapon, danger; moderate: hurt, harm, threat; faint: risk |
| `reward` | Gray RST, operant | Reinforcement | strong: triumph, rewarded, victory; moderate: success, earned, win; faint: found, received |
| `exclusion` | Williams (2007) ostracism | Social exclusion | strong: ostracized, shunned, banished; moderate: excluded, isolated, marginalized; faint: left out, rejected |

### 12.3 Key Constraints (Repeated for Visibility)

1. **Literature-rooted.** Every vocabulary term traceable to the construct literature that defines the INPUT node. No intuitive keyword lists.
2. **Weighted, not flat.** Terms carry activation degrees (0.0-1.0). "Humiliated" activates `dominance` at 0.8; "stumbled" at 0.3.
3. **Universal across characters.** Same vocabulary for everyone. Personality differences emerge from connection weights and drive amplification, not from different input features (Mischel & Shoda 1995; Read & Miller 2002).
4. **Additive layers.** Include all three layers (vocabulary, substrate, drive amplification). Do not remove any signal without empirical proof of redundancy.
5. **Empirically validated.** Classify existing transcripts with the new vocabulary. Check which nodes fire. Compare to human judgment. Tune weights based on results (D4a).

### 12.4 Memory-to-Catalogue Mapping Methodology

**When to use:** whenever mapping a formation memory to a catalogue entry in `experience-behaviour-catalogue/`. This methodology was developed during Layer B and should be applied consistently for any new characters or memory additions.

#### The Selection Principle

The `serves` annotation and the `catalogue` entry are complementary:
- **`serves`** says WHICH DRIVE the memory calibrates — a structural tag for testing and salience
- **`catalogue`** says WHICH PSYCHOLOGICAL MODEL explains HOW the memory calibrates the MEDIATING NODES

They answer different questions. Do not conflate them.

#### The Three Tests

Every mapping must pass:

1. **Directional correctness.** Does the catalogue entry's `expected_outcomes` calibrate the RIGHT mediating nodes in the RIGHT direction for this character? Check each node in the entry's outcomes against the character's target substrate.

2. **Cumulative trajectory.** Does the sequence of catalogue entries across ALL memories produce the extreme substrate §10.8 requires? Trace each mediating node across the full memory sequence and verify the cumulative calibration is sufficient (e.g., DD needs self_worth ↓↓, not just ↓).

3. **No unresolvable contradictions.** Do any outcomes CONTRADICT the character's profile? Mild conflicts (DD: cbt-negative-self → withdraw ↑, counteracted by mastery → approach ↑) are acceptable — the CAPS network resolves them through attractor dynamics. Strong conflicts (Pen: attachment-anxious → other_reliability ↓ for a secure-attachment character) are not — they push the substrate in the wrong direction.

#### The OUTPUT-vs-MEDIATING Rule

**Always prefer catalogue entries that calibrate MEDIATING nodes over those that only affect OUTPUT nodes.**

Mediating nodes: `self_worth`, `self_efficacy`, `other_reliability`, `other_safety`, `threat_sensitivity`, `FFFS_activation`, `BAS_activation`, `reinforcement_expectation`, `arousal_level`, `escape_assessment`, `distrust`, `self_direct`, `trust`

OUTPUT nodes: `approach`, `persist`, `withdraw`, `fight`, `flight`, `freeze`, `assert`, `comply`, `submit`, `explore`, `rebel`, `fawn`, `abandon`, `depend`

Many Bandura and operant entries only calibrate OUTPUT nodes. These are valid catalogue entries but weak Layer B choices because Break #4 is specifically about mediating node calibration. Entries like `bandura-observational-learning` (approach ↑, persist ↑, generalize ↑ — all OUTPUT) should be rejected in favour of entries that move the mediating substrate.

**Exception:** when no mediating-node entry fits the memory content, an OUTPUT-only entry is acceptable as a last resort. Document why in the mapping rationale.

#### Applying to a New Character

1. Define the **target mediating state** first: which mediating nodes need extreme values for the character's drives to emerge? Reference §10.8 (intensity principle) and §10.9 (emergent drives).

2. For each memory in chronological order, ask: "which catalogue entry's `expected_outcomes` move the mediating nodes toward the target state?"

3. Verify the **escalation trajectory** (§10.8): successive memories should push mediating nodes FURTHER, not reset them. If a memory reinforces a previous calibration, using the same catalogue entry twice is correct (e.g., DD's two `cbt-negative-self` entries for escalating recognition deprivation).

4. Check for **attachment profile consistency**: the character's declared attachment style (secure, dismissive, anxious, fearful) should be consistent with the attachment catalogue entries used. Don't use `attachment-anxious` for a secure character unless the narrative specifically justifies an anxiety episode.

5. Verify that the **character's known behavioral profile** isn't contradicted: if the character is persistent and approach-motivated, don't use entries with `persist` ↓ or `withdraw` ↑ unless other entries compensate.

#### Rejected Mappings — Lessons Learned

| Rejected Entry | Problem | Lesson |
|----------------|---------|--------|
| `cbt-negative-world` for HC age 15 | withdraw ↑, persist ↓ contradict HC's cold patience | Check ALL expected_outcomes, not just the thematic fit |
| `attachment-anxious` for Pen age 7 | other_reliability ↓, self_worth ↓ contradict secure attachment | Attachment entries must match the declared attachment style |
| `operant-negative-reinforcement` for DD age 6 | flight ↑, withdraw ↑ — DD takes, he doesn't flee | Match the BEHAVIORAL mechanism, not just the emotional tone |
| OUTPUT-only entries (bandura-observational-learning, bandura-model-imitation, operant-positive-reinforcement) as primary choices | No mediating node calibration | Layer B exists to calibrate mediating nodes — OUTPUT-only entries miss the target |

### 12.5 Memory-to-Catalogue Mapping (Layer B)

The authoritative mapping. Each formation memory → catalogue entry, with the mediating nodes it calibrates.

**DD (Dick Dastardly)** — Target: self_worth ↓↓, self_efficacy ↑, other_reliability ↓

| Age | Serves | Catalogue | Key Mediating Calibration |
|-----|--------|-----------|--------------------------|
| 6 | greed\|defining, scheming\|moderate | `attachment-dismissive` | other_reliability ↓, distrust ↑, self_direct ↑ |
| 8 | recognition\|defining | `cbt-negative-self` | self_worth ↓ |
| 10 | scheming\|strong | `bandura-mastery-efficacy` | self_efficacy ↑ |
| 12 | gloating\|defining | `operant-positive-reinforcement` | approach ↑, persist ↑ |
| 14 | scheming\|strong, recognition\|strong | `cbt-negative-self` | self_worth ↓ (reinforced) |

**HC (Hooded Claw)** — Target: other_reliability ↓↓↓↓, distrust ↑↑↑↑

| Age | Serves | Catalogue | Key Mediating Calibration |
|-----|--------|-----------|--------------------------|
| 6 | self-preservation\|defining | `attachment-dismissive` | other_reliability ↓, distrust ↑, self_direct ↑ |
| 9 | self-preservation\|strong | `cbt-negative-other` | other_reliability ↓, other_safety ↓, distrust ↑ |
| 12 | dominance\|strong, scheming\|moderate | `bandura-mastery-efficacy` | self_efficacy ↑ |
| 14 | dominance\|strong | `cbt-negative-other` | other_reliability ↓, other_safety ↓, distrust ↑ |
| 15 | scheming\|defining | `attachment-trust-erosion` | distrust ↑↑, other_reliability ↓↓ |
| 18 | scheming\|strong | `operant-autonomy-reinforcement` | self_direct ↑, rebel ↑ |

**PP (Peter Perfect)** — Target: self_efficacy ↑↑ (external), other_reliability ↑↑

| Age | Serves | Catalogue | Key Mediating Calibration |
|-----|--------|-----------|--------------------------|
| 5 | protection\|moderate, proving-worth\|moderate | `attachment-secure` | self_worth ↑, other_reliability ↑, trust ↑ |
| 8 | proving-worth\|defining | `bandura-mastery-efficacy` | self_efficacy ↑ |
| 12 | protection\|moderate, proving-worth\|moderate | `cbt-positive-other` | other_reliability ↑, other_safety ↑, trust ↑ |
| 14 | proving-worth\|strong | `bandura-mastery-efficacy` | self_efficacy ↑ (reinforced) |
| 16 | gallantry\|strong, proving-worth\|strong | `cbt-positive-self` | self_worth ↑ |

**Pen (Penelope)** — Target: other_safety ↑↑, self_efficacy ↑↑

| Age | Serves | Catalogue | Key Mediating Calibration |
|-----|--------|-----------|--------------------------|
| 5 | curiosity\|defining | `bandura-mastery-efficacy` | self_efficacy ↑ |
| 7 | social-harmony\|defining | `attachment-secure` | other_safety ↑, other_reliability ↑, trust ↑ |
| 9 | social-harmony\|moderate, adventure\|moderate | `cbt-positive-other` | other_safety ↑, other_reliability ↑, trust ↑ |
| 11 | adventure\|defining | `bandura-mastery-efficacy` | self_efficacy ↑ (reinforced) |
| 13 | social-harmony\|strong (×2) | `cbt-positive-self` | self_worth ↑ |

**AHM (Ant Hill Mob)** — Target: other_reliability CONFLICTED, distrust ↑↑

| Age | Serves | Catalogue | Key Mediating Calibration |
|-----|--------|-----------|--------------------------|
| 8 | loyalty\|defining | `attachment-secure` | other_reliability ↑, trust ↑, self_worth ↑ |
| 10 | suspicion\|defining | `trauma-hypervigilance` | distrust ↑, cautious_approach ↑ |
| 12 | protection\|defining | `cbt-positive-self` | self_worth ↑ |
| 14 | loyalty\|strong | `attachment-earned-security` | trust ↑, other_reliability ↑ |
| 15 | suspicion\|strong | `attachment-trust-erosion` | distrust ↑↑, other_reliability ↓↓ |

---

## 13. File Locations

| What | Path |
|---|---|
| Social config (GENERIC) | `examples/wacky-manor/src/main/resources/META-INF/eidos/social-config-generic.yaml` |
| FormationMemoryParser | `examples/wacky-manor/src/main/java/.../agent/FormationMemoryParser.java` |
| FormationMemoryParserTest | `examples/wacky-manor/src/test/java/.../agent/FormationMemoryParserTest.java` |
| SocialConfig / FormationMemory | `examples/wacky-manor/src/main/java/.../agent/SocialConfig.java` |
| ManorSocialConfigLoader | `examples/wacky-manor/src/main/java/.../agent/ManorSocialConfigLoader.java` |
| CharacterCognition | `examples/wacky-manor/src/main/java/.../agent/CharacterCognition.java` |
| ManorCognitiveSeeder | `examples/wacky-manor/src/main/java/.../agent/ManorCognitiveSeeder.java` |
| DriveExpressionEvalTest | `examples/wacky-manor/src/test/java/.../experiment/DriveExpressionEvalTest.java` |
| Phase 0 eval results | `examples/wacky-manor/docs/eval/phase0-audit-baseline-2026-10-13/RESULTS.md` |
| Phase 0 micro test scores | `examples/wacky-manor/docs/eval/phase0-audit-baseline-2026-10-13/micro-test-scores.json` |
| Phase 0 run 1 transcript | `examples/wacky-manor/docs/eval/phase0-audit-baseline-2026-10-13/transcript.json` |
| Phase 0 run 2 transcript | `examples/wacky-manor/docs/eval/phase0-fix-run-2026-10-13/transcript.json` |
| Design spec (drive integration) | `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-09-drive-integration-architecture-design.md` |
| Design spec (memory concern) | `wsp-casehub/specs/issue-115-drive-expression-gap/2026-10-13-formation-memory-concern-architecture-design.md` |
| Decisions | `wsp-casehub/specs/issue-115-drive-expression-gap/decisions.md` |
| CAPS topology (authoritative) | `neocortex/caps-engine/src/main/resources/caps-topology.yaml` |
| CapsNode (API) | `neocortex/caps-api/src/main/java/.../caps/CapsNode.java` |
| CapsTopologyLoader | `neocortex/caps-engine/src/main/java/.../caps/engine/CapsTopologyLoader.java` |
| RuleBasedSituationClassifier | `neocortex/caps-engine/src/main/java/.../caps/engine/RuleBasedSituationClassifier.java` |
| RuleBasedSituationClassifierTest | `neocortex/caps-engine/src/test/java/.../caps/engine/RuleBasedSituationClassifierTest.java` |
| CapsTopologyLoaderTest | `neocortex/caps-engine/src/test/java/.../caps/engine/CapsTopologyLoaderTest.java` |
| LlmSituationClassifier | `neocortex/caps-llm-classifier/src/main/java/.../caps/llm/LlmSituationClassifier.java` |
| Catalogue | `neocortex/docs/specs/experience-behaviour-catalogue/` |
| ConcernBasedSalienceStrategy | `neocortex/cognition/src/main/java/.../salience/ConcernBasedSalienceStrategy.java` |
| FormativeExperience | `neocortex/memory-api/src/main/java/.../experience/FormativeExperience.java` |
| classify_emotions.py | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| Blog entries | `wsp-casehub/blog/2026-10-13-mdp01-when-your-characters-stop.md`, `2026-10-13-mdp02-from-authoring-to-architecture.md` |
