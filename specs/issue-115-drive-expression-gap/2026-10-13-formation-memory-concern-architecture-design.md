# Formation Memory Concern Architecture — Unifying Memories, Concerns, and Salience

**Date:** 2026-10-13
**Extends:** 2026-10-09-drive-integration-architecture-design.md (D1-D8)
**Scope:** #115 (formation memory audit + `serves` annotation system + testing pyramid)
**Branch:** issue-115-drive-expression-gap
**Covers:** #115

---

## 1. Problem

Formation memories are the strongest behavioral signal in the LLM prompt — research shows that few-shot demonstrations override competing personality instructions (ACL 2025, Persona-Aware Contrastive Learning). Yet our formation memories:

1. **Pre-resolve drive conflicts** — the author decides which drive dominates ("the rush wasn't cruelty — it was the architecture"), preventing the personality model from doing its job
2. **Prime specific actions** — descriptions of what the character DOES ("you practiced on the gardener next") rather than situations the personality model interprets
3. **Have no structural link to the cognitive pipeline** — seeded as `FormativeExperience` with placeholder `catalogueEntryId: "formation-memory"` and generic `situationTypes: ["formation", period]`
4. **Cannot be tested in isolation** — no static consistency checking, no micro unit tests, full scenario runs (30+ min) are the only validation
5. **Are all active simultaneously** — unlike human autobiographical memory (Conway 2005), where the working self selectively activates goal-relevant memories

This design adds the `serves` annotation system to connect formation memories to the drive-concern derivation chain, enable micro testing, and lay the foundation for selective memory activation.

## 2. Theoretical Foundations

### 2.1 Two-Layer Memory-Concern Association

Two empirically established mechanisms operate at different levels:

**Structural association (Singer 2013):** Self-defining memories have stable, enduring links to concerns. "The pool of self-defining memories possesses a synchronic existence within narrative identity; they remain relatively static." A DD memory about being unwitnessed is ALWAYS linked to the recognition concern — this doesn't change with context.

**Dynamic activation (Conway 2005, Klinger 1971):** WHICH structurally-linked memories get activated depends on the current working self's goals. "The working self modulates access to long-term knowledge... episodic memories relevant to the goal of the extant working self are activated through direct retrieval." When recognition is currently high-intensity, recognition-linked memories surface more readily.

The `serves` annotation captures the structural layer. D5's PAD-proximity/Klinger salience captures the dynamic layer. Neither replaces the other.

### 2.2 Concern Vocabulary

Singer (2013) and Klinger (1971) establish that concerns are individual-specific, not universal categories. McClelland's three needs (achievement, affiliation, power) and SDT's three (autonomy, competence, relatedness) are too coarse — they collapse distinct behavioral signatures.

Drive type names ARE concerns in Klinger's sense — specific motivational goal-pursuits. D1 already maps each drive type to CAPS nodes with literature sourcing. Higher-level themes (insecurity, need-for-validation) are derivable compositions of drive types + facet configurations that emerge from the graph of annotations rather than requiring a separate vocabulary.

### 2.3 Memory Functions (Pillemer 2003, Bluck 2003)

Autobiographical memories **serve** three functions: directive (guide behavior), self (maintain identity), social (bond with others). The term "serves" is the established verb for the memory-concern relationship across the Pillemer, Bluck, and Singer literature.

The degree scale maps to Pillemer's directive function taxonomy:
- **faint** — memory touches the concern peripherally
- **moderate** — memory engages the concern but isn't defined by it
- **strong** — concern is a dominant force in the memory
- **defining** — the memory IS the formation of this concern (Pillemer's "originating event")

## 3. Architecture

### 3.1 Derivation Chain

Extends the 2026-10-09 design spec with the `serves` annotation layer:

```
Personality Model (AMPD facets + drives + attachment)
  ↓ [D1 mappings — literature-sourced per drive type]
CAPS Network (self_worth, BAS_activation, threat_sensitivity, ...)
  ↓ [derive — facet + drive intensity combinations]
Enduring Concerns (drive types as Klinger concerns)
  ↓ [D10 structural link — Singer SDMs, `serves` annotations]
Formation Memories (annotated with {{serves: drive|degree: text}})
  ↓ [D10 dynamic activation — Conway SMS / D5 PAD-proximity]
Activated Memory Subset (salience-ordered by current concern state)
  ↓ [D6 rendering strategies]
LLM Context → Emergent Behavior
```

Each arrow is independently testable (see §4 Testing Pyramid).

### 3.2 The `serves` Annotation

#### Inline syntax

```yaml
formation-memories:
  - age: 14
    episode: >-
      You built an elaborate trap for the neighbourhood cat — a system
      of strings, counterweights, and a bucket. {{serves: scheming|strong:
      When it worked perfectly, there was a settling in your chest — the
      pieces had come together exactly as you'd mapped them.}} Nobody else
      saw it. {{serves: recognition|strong: It felt so much better when
      there was someone to show it to — the architecture deserved an
      audience.}}
    pleasure: 0.4
    arousal: 0.5
    dominance: 0.7
```

#### Whole-memory form (simple cases)

```yaml
  - age: 6
    serves: scheming|defining
    episode: >-
      The first time you hid a bread roll in your pocket...
```

#### Parser behavior

1. **Extract** — regex `\{\{serves:\s*(\w[\w-]*)\|(\w+):\s*(.*?)\}\}` captures drive name, degree, and annotated text
2. **Strip** — for prompt rendering, replace each `{{serves: ...: text}}` with just `text`. LLM receives clean episode narrative.
3. **Store** — annotations as structured metadata on `FormationMemory`:

```java
public record FormationMemory(
    int age, String episode,
    double pleasure, double arousal, double dominance,
    List<ConcernAnnotation> annotations  // NEW — extracted from episode text
) {
    public record ConcernAnnotation(
        String drive, String degree, String text,
        int startOffset, int endOffset
    ) {}
}
```

4. **Expose** — annotations available for Level 0 checker and Level 2 test generation

#### Vocabulary

`serves` references drive type names from social-config YAML. These connect to CAPS nodes via D1 mappings:

| Drive Type | CAPS Nodes (from D1) | Literature |
|---|---|---|
| gallantry | other_safety, acceptance, approach | Batson 1991 |
| proving-worth | mastery, competence_recognition, self_efficacy | McClelland 1961 |
| protection | other_safety, physical_threat, approach | George & Solomon 2008 |
| scheming | dominance, strategic_cognition, distrust | Christie & Geis 1970 |
| recognition | competence_recognition, self_worth, BAS_activation | McClelland 1961 |
| gloating | dominance, BAS_activation | TBD — Tier 2/3 |
| self-preservation | threat_sensitivity, escape_assessment, FFFS_activation | Gray & McNaughton 2000 |
| curiosity | explore, BAS_activation | Berlyne 1960 |
| social-harmony | other_safety, acceptance, proximity_seek | SDT relatedness |
| adventure | explore, BAS_activation, approach | Zuckerman 1994 |
| greed | BAS_activation, approach, persist | McClelland nAch variant |
| loyalty | other_reliability, proximity_seek, trust | Bowlby attachment |
| suspicion | threat_sensitivity, distrust, BIS_activation | BIS/BAS RST |

#### Degree scale

| Degree | Meaning | Pillemer equivalent |
|---|---|---|
| faint | Peripherally touches the concern | Incidental reference |
| moderate | Engages the concern but isn't defined by it | Supporting episode |
| strong | Concern is a dominant force in the memory | Central experience |
| defining | Memory IS the formation of this concern | Originating event |

### 3.3 Connection to Seeding Pipeline

`ManorCognitiveSeeder.seedFormationMemories()` currently sets:
- `catalogueEntryId: "formation-memory"` (placeholder)
- `situationTypes: ["formation", period]` (generic)

With `serves` annotations, the seeder gains:
- `situationTypes` derived from the annotated drive types (e.g., `["formation", "childhood", "scheming", "recognition"]`)
- `metadata` enriched with annotation data for downstream retrieval

This connects formation memories to the Klinger concern-based salience strategy — `ConcernBasedSalienceStrategy` matches observation text against active concerns, and annotated memories carry concern-relevant keywords from their drive type mappings.

### 3.4 Connection to Catalogue

Formation memories with `serves` annotations can optionally reference catalogue entries:

```yaml
  - age: 9
    serves: suspicion|defining
    catalogue: attachment-dismissive
    episode: >-
      A boy at school shared his lunch with you because you looked sad.
      You felt uncomfortable.
```

The `catalogue` field connects the memory to the Experience-to-Behaviour Catalogue entry, enabling:
- Expected outcome verification (does this memory's PAD match the catalogue's expected_outcomes?)
- Cross-model consistency (does the memory's drive annotation align with the catalogue entry's model?)

This is optional — not every formation memory maps to a catalogue entry.

## 4. Testing Pyramid

### Level 0 — Static Consistency (milliseconds, no LLM)

A Python or Java checker that:
1. Parses `social-config-generic.yaml`
2. Extracts all `serves` annotations (inline and whole-memory)
3. For each annotation, verifies:
   - The drive name exists in the character's drive list
   - The annotated text is consistent with the drive's CAPS node mapping (keyword check)
   - No contradiction with the character's facet scores (e.g., a memory serving "recognition" for a character with attention-seeking:5 would be flagged)
   - No pre-resolved drive conflicts ("not X, it was Y" patterns)
   - No action priming (descriptions of specific behaviors rather than situations/feelings)

Runs on every commit. Fails the build if contradictions are found.

### Level 1 — Prompt Rendering Diagnostic (seconds, no LLM)

Per-character diagnostic test (extends existing `ppDiagnostic_dumpCognitiveContext`):
1. Renders cognitive sections for each character
2. Verifies all drives present in "What Pulls at You"
3. Verifies formation memories render in "Where You Come From" with annotations stripped
4. Prints the complete prompt context for manual inspection

### Level 2 — Single-Situation Drive Expression (~10s, LLM)

For each collapsed drive, one micro test:
1. Present a situation designed to activate the target drive
2. Run the character through the cognitive pipeline
3. Judge whether the drive expresses in thinking/dialogue
4. Score 0-5 on drive expression AND emergence (not scripted)

Pattern: `DriveExpressionEvalTest` (already exists for PP proving-worth).

New tests needed for: HC gloating, DD gloating, PP gallantry, Penelope social-harmony, Penelope adventure.

### Level 3 — Drive Differentiation (~10s, LLM)

For drive pairs that are confused (PP gallantry vs proving-worth):
1. Present a situation that should activate drive A but not B
2. Judge which drive dominates the response
3. Verify the personality model differentiates them

### Level 4 — If-Then Behavioral Signature (~1min, LLM)

For each character, 5-6 situations mapping to different drives:
- Situation 1 → should activate drive A
- Situation 2 → should activate drive B
- etc.

Run all, verify the pattern matches CAPS predictions. This is Mischel's behavioral signature — the if-then profile that IS the personality.

### Level 5 — Full Scenario (~30min, LLM)

Only after Levels 0-4 pass. Full scenario run + classify_emotions.py. Confirms micro improvements compose into macro improvement.

## 5. Memory Audit Principles

All existing formation memories are audited against these rules, derived from D4 (emergence philosophy) and the LLM persona research finding that demonstrations override instructions:

### 5.1 No pre-resolved drive conflicts

**Remove:** "not X, it was Y" patterns where the author decides which drive dominates.

Example — DD age 14:
> Before: "That felt wrong — like a firework going off in an empty field."
> After: "{{serves: recognition|strong: It felt so much better when there was someone to show it to — the architecture deserved an audience.}}"

The original pre-resolves to "wrong" (recognition frustration dominates). The revision uses comparative framing that preserves scheming satisfaction while engaging recognition.

### 5.2 No action priming

**Remove:** descriptions of what the character DOES, which the LLM pattern-matches to.

Example — HC age 12:
> Before: "You practiced on the gardener next. Then the cook."
> This tells the LLM to have HC practice manipulation on multiple targets. The scheming drive + manipulativeness:85 facet should produce this behavior emergently.

### 5.3 No value declarations

**Remove:** statements that resolve what things mean.

Example — HC age 14:
> Before: "you stopped wanting to be liked. You wanted to be owed"
> This resolves the dominance vs affiliation drive conflict. The personality model (callousness:90, grandiosity:80, intimacy-avoidance:75) should derive this shift.

### 5.4 Emotional texture is allowed

Memories CAN describe how things felt, as long as they don't pre-resolve which drive produced the feeling. The felt experience IS the memory (Conway 2000) — stripping all emotion makes characters flat.

Example — HC age 15:
> "Not anger. Something colder. Something patient."
> This describes a felt quality without naming a drive. Borderline acceptable — it points toward scheming but doesn't declare it. Mark with `{{serves: scheming|defining: ...}}`.

### 5.5 `serves` annotations document intent

Every memory gets annotated with which drives it serves and at what degree. The audit is then testable — Level 0 checker verifies consistency.

## 6. Implementation Phasing

This design extends the existing phasing (2026-10-09 spec §5) with a new Phase 0:

| Phase | What | Validates | Gate |
|---|---|---|---|
| **0a** | Add `serves` to FormationMemory record, update loader and parser | Annotations extracted, prompt text clean | Unit test: parse → extract → strip roundtrip |
| **0b** | Annotate all formation memories in social-config-generic.yaml with `serves` | Every memory has at least one annotation | Level 0 checker passes |
| **0c** | Audit and fix memories per §5 principles | No contradictions, no pre-resolutions, no action priming | Level 0 checker passes with audit rules |
| **0d** | Build Level 2-4 micro tests for 5 collapsed drives | Each drive has a testable situation | Tests run, baseline scores captured |
| **0e** | Run full scenario (Level 5) to establish post-audit baseline | Baseline score captured | Compare to L4b (mean 3.82) |
| 1-5 | Per existing spec (drive→CAPS, drive→appraisal, drive→memory salience, enrich, remove somatic) | Per existing spec | Per existing spec |

Phase 0 is all in `examples/wacky-manor` — no cross-repo changes needed. The annotation system, audit, and testing infrastructure are self-contained in the examples repo until Phase 3 (drive→memory salience) connects them to the neocortex runtime.

## 7. Cross-Repo Impact

| Repo | Phase | Changes |
|---|---|---|
| `examples/wacky-manor` | 0a-0e | FormationMemory record, loader, parser, social-config-generic.yaml, Level 0 checker, Level 2-4 tests |
| `neocortex/memory-api` | 3 | FormativeExperience: situationTypes from annotations |
| `neocortex/cognition` | 3 | ConcernBasedSalienceStrategy: annotation-aware matching |
| catalogue | — | No changes — vocabulary extends naturally via new drive→CAPS mappings |

## References

### Primary Theory (new to this spec)
- Singer, J.A., Blagov, P., Berry, M. & Oost, K.M. (2013). Self-Defining Memories, Scripts, and the Life Story. *Journal of Personality*, 81(6), 569-582.
- Conway, M.A. (2005). Memory and the Self. *Journal of Memory and Language*, 53(4), 594-628.
- Conway, M.A. & Pleydell-Pearce, C.W. (2000). The Construction of Autobiographical Memories in the Self-Memory System. *Psychological Review*, 107(2), 261-288.
- Conway, M.A., Singer, J.A. & Tagini, A. (2004). The Self and Autobiographical Memory: Correspondence and Coherence. *Social Cognition*, 22(5), 491-529.
- Pillemer, D.B. (2003). Directive Functions of Autobiographical Memory. *Memory*, 11(2), 193-202.
- Bluck, S. (2003). Autobiographical Memory: Exploring its Functions in Everyday Life. *Memory*, 11(2), 113-123.
- McClelland, D.C. (1961). *The Achieving Society*. Van Nostrand.
- Murray, H.A. (1938). *Explorations in Personality*. Oxford UP.
- Deci, E.L. & Ryan, R.M. (2000). Self-Determination Theory and the Facilitation of Intrinsic Motivation. *American Psychologist*, 55(1), 68-78.

### LLM Persona Research
- ACL 2025: Persona-Aware Contrastive Learning for LLM Role-Playing
- Han et al. (2026): Personality Expression Across Contexts in LLM Agents. arXiv:2602.01063.

### Inherited References
- See 2026-10-09-drive-integration-architecture-design.md §References for CAPS, Scherer, Klinger, and drive-specific mapping sources.

### Decisions
- D9: Formation Memory Concern Vocabulary
- D10: Two-Layer Memory-Concern Association
- D11: Annotation Syntax and Degree Scale
- D1-D8: See 2026-10-09 spec
