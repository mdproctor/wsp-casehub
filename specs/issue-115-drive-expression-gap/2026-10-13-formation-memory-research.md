# Formation Memory Research — Complete Reference

**Date:** 2026-10-13
**Purpose:** Internal research document for session continuity. Covers literature findings, implementation mapping, audit results, and iteration plan. A future session should be able to read ONLY this document and have full context for the next iteration.
**Branch:** issue-115-drive-expression-gap

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

## 10. File Locations

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
| CAPS topology | `neocortex/docs/specs/2026-10-02-caps-topology.yaml` |
| Catalogue | `neocortex/docs/specs/experience-behaviour-catalogue/` |
| ConcernBasedSalienceStrategy | `neocortex/cognition/src/main/java/.../salience/ConcernBasedSalienceStrategy.java` |
| FormativeExperience | `neocortex/memory-api/src/main/java/.../experience/FormativeExperience.java` |
| classify_emotions.py | `examples/wacky-manor/docs/eval/classify_emotions.py` |
| Blog entries | `wsp-casehub/blog/2026-10-13-mdp01-when-your-characters-stop.md`, `2026-10-13-mdp02-from-authoring-to-architecture.md` |
