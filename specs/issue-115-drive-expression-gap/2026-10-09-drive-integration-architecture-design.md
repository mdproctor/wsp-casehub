# Drive Integration Architecture — Reconnecting Disconnected Cognitive Subsystems

**Date:** 2026-10-09
**Scope:** #118 (drive→CAPS, drive→appraisal, drive→memory salience), #119 (Klinger's current concerns theory)
**Branch:** issue-115-drive-expression-gap
**Covers:** #115, #118, #119

---

## 1. Problem

Three cognitive subsystems adapt or produce behavioral signals independently, with no cross-talk:

```
Drive Adaptation ──→ updates intensities ──→ NOTHING READS THEM
CAPS Settling    ──→ behavioral attractors ──→ renders (but drive-blind)
Appraisal        ──→ situational emotions  ──→ renders (but drive-blind)
Formation Memories ──→ static rendering    ──→ no salience modulation
```

A character's proving-worth drive can intensify from 0.7 → 0.95 over 300 events of reinforcement and it changes absolutely nothing about what the LLM sees. The adaptation is computation with no output. The systems are disconnected by design accident — built at different times to solve different problems, never reconciled.

**Prior failure:** Issue #115 showed that removing prescriptive text (somatic drive descriptions) caused drive expression to collapse — gloating dropped -1.52 (from 3.97 to 2.45). The LLM could not infer drive-appropriate behavior from drive labels and intensities alone. This confirmed that computational enrichment must replace prescriptive text before it can be removed.

**Root cause:** The cognitive architecture partially implements established psychological models but leaves key connections unwired. Scherer's appraisal model prescribes that motivational state modulates relevance detection — but character drives were never connected to the SEC checks. Mischel & Shoda's CAPS theory prescribes that goals/motivations participate as activations in the cognitive-affective network — but drives were never injected into CAPS settling. Klinger's current concerns theory predicts that active goal pursuits increase salience for concern-related stimuli — but no salience modulation exists for formation memories.

The fix is not more prescriptive text. The fix is wiring the connections the theory predicts.

## 2. Theoretical Foundations

### 2.1 CAPS — Cognitive-Affective Processing System (Mischel & Shoda, 1995)

Personality as a network of cognitive-affective units (CAUs) with stable connection weights. Behavior patterns emerge from network settling dynamics — the "behavioral signatures" that distinguish individuals. Five CAU types: encodings, expectancies, affects, goals/motivating forces, competencies.

**Key principle for this design:** Goals and motivating forces are one of the five CAU types. In computational CAPS models (Read & Miller, 2002; Read et al., 2018), goals operate as **activations** that flow through the weighted network to bias behavior selection. Connection weights represent stable personality traits from developmental history. Weights change through long-term learning; activations change with current motivational state.

**Application:** Character drives (gallantry, proving-worth, protection) are persistent activations injected into the CAPS network alongside transient situation activations. They bias the network's settling dynamics toward drive-congruent behavioral attractors. The behavior emerges from the network — it is not prescribed.

### 2.2 Scherer's Component Process Model (Scherer, 2001)

Emotional appraisal through sequential evaluation checks (SECs): relevance, implication, coping potential, normative significance. The relevance SEC explicitly evaluates **goal/need relevance** — "its motivational consistency, i.e., its conduciveness to satisfy a need, reach a goal, or uphold a value."

**Key principle:** Events relevant to active goals/drives produce stronger emotional responses (Lazarus, 1991; Smith & Kirby, 2009). Motivational state is a first-class input to appraisal, not an optional modulator.

**Application:** Character drives flow through the existing SEC checks via `DriveCategory.CHARACTER`. Events relevant to a character's strong drives appraise as more emotionally significant. A high-protection character reacts more intensely to threat events.

### 2.3 Klinger's Current Concerns Theory (Klinger, 1971; Klinger & Cox, 2004)

Active motivational states ("current concerns") increase cognitive salience for concern-related stimuli. This operates across perception, attention, memory retrieval, and emotional reactivity. The effect is proportional to goal commitment.

**Key principle:** When a drive is active and strong, stimuli related to that drive become more salient — more likely to be noticed, retrieved, and emotionally processed.

**Application:** Drive intensities modulate which formation memories surface. Memories whose affective profile (PAD values) aligns with active drives become more salient. This is operationalized via PAD proximity (Mehrabian, 1996), validated by mood-congruent memory research (Bower, 1981) and dimensional memory retrieval studies (Rubin & Talarico, 2009).

### 2.4 PAD Model (Mehrabian, 1996)

Pleasure-Arousal-Dominance as a dimensional framework for temperament — not just momentary emotion. PAD explains ~75% of reliable variance in Big Five personality factors. Validated for stable motivational dispositions, not only emotional states.

**Application:** Formation memories have explicit PAD values. Drives have PAD centroids (seeded from initial config, evolved through reinforcement history). PAD proximity computes association strength between memories and drives. Valence weighted more heavily than arousal/dominance (PMC mood-congruent memory review, 2023).

## 3. Architecture

### 3.1 Three Activation Types in the CAPS Network

| Type | Source | Persistence | Represents |
|------|--------|-------------|------------|
| Connection weights | disposition_modifiers (config) | Stable | Personality traits |
| Situation activations | SituationClassifier.classify() | Transient (per-event) | Current events |
| Drive activations | DriveAdaptationPhase (NEW) | Persistent (re-injected each tick) | Motivational bias |

Drive activations are added to `combinedActivations` in `BehavioralSynthesisPhase` before `capsEngine.settle()`. Merge strategy: additive with configurable dampening. Existing activation clamping [-1, 1] prevents saturation.

### 3.2 Integration Point 1: Drive → CAPS (Persistent Dispositional Activations)

**What changes:** `BehavioralSynthesisPhase.run()` reads drive-intensity nodes from the cognitive subgraph (already produced by `DriveAdaptationPhase` at priority 17) and converts them to CAPS activations before settling.

**Drive-to-activation mapping:** Each drive type maps to CAPS nodes it activates, defined universally (same for all characters). Mappings derived from published motivational psychology literature:

| Drive Type | Literature Source | CAPS Activations |
|------------|-------------------|------------------|
| Gallantry/prosocial | Batson (1991), Penner et al. (2005) | `other_safety`, `acceptance`, `approach` |
| Proving-worth/achievement | McClelland (1961) | `mastery`, `competence_recognition`, `self_efficacy` |
| Protection/caregiving | George & Solomon (2008) | `other_safety`, `physical_threat`, `approach` |
| Scheming/Machiavellianism | Christie & Geis (1970) | `dominance`, `social_cognition`, `distrust` |
| Gloating/schadenfreude | Literature TBD — Tier 2/3 | TBD |
| Self-preservation | Literature TBD — Tier 2/3 | TBD |

Each mapping is Tier 2 sourcing (D2) — derived from published research with documented derivation chain. Mappings for remaining drive types require literature review during implementation.

**Activation formula:** `activation = mapping_weight × drive_intensity × dampening_factor`

**Configuration:**
- `manor.caps.drive-dampening` (default 0.3) — prevents drive activations from overwhelming situation activations
- `manor.caps.drive-activation-enabled` (default true) — master toggle for micro experiments

**Key files:**
- `neocortex/mindmap-intelligence/.../BehavioralSynthesisPhase.java` — add drive activation reading before `capsEngine.settle()`
- `neocortex/caps-api/.../CapsEngine.java` — no change needed (settle() already takes `Map<String,Double>`)
- New: drive-type activation mapping config (YAML alongside caps-topology or in a separate `drive-caps-mapping.yaml`)

### 3.3 Integration Point 2: Drive → Appraisal (Call-Site Merge)

**What changes:** `AppraisalTickParticipant.tick()` reads character drives from MindMap drive-intensity nodes and merges them with SDT drives before constructing `AppraisalContext`.

**Wiring:**
1. `AppraisalTickParticipant` gains `MindMapStore` injection
2. On tick: query MindMap for `cognitiveKind=drive-intensity` nodes for the agent
3. Convert each to `appraisal.Drive` with `DriveCategory.CHARACTER`
4. Merge with SDT drives from `driveOrchestrator.currentDrives()`
5. Pass merged list to `AppraisalContext`

**Relevance detection:** Character drives use keyword lists (not drive-name substring matching). The `Drive` record's trigger field is extended from `String` to `List<String>` to hold situationally-relevant terms:

```yaml
# In social-config.yaml, alongside drive definitions
drives:
  - type: gallantry
    intensity: 0.9
    triggers: ["help", "courtesy", "rescue", "protect", "defend", "kindness"]
  - type: scheming
    intensity: 0.8
    triggers: ["plan", "trap", "ambush", "outwit", "scheme", "trick", "deceive"]
```

`RelevanceCheck.computeDriveRelevance()` matches these keywords against event narratives using the same TF-style scoring pattern as `RuleBasedSituationClassifier`.

**Configuration:**
- `manor.appraisal.character-drive-weight` (default 1.0) — scaling factor for character drive relevance
- `manor.appraisal.character-drives-enabled` (default true) — master toggle

**Key files:**
- `neocortex/cognition/.../appraisal/AppraisalTickParticipant.java` — add MindMapStore injection, merge drives
- `neocortex/cognition-api/.../appraisal/Drive.java` — extend trigger to `List<String>`
- `wacky-manor/.../social-config.yaml` — add `triggers` field to drive entries

### 3.4 Integration Point 3: Drive → Memory Salience (#119 — Klinger's Current Concerns)

**What changes:** `CharacterCognition.renderFormationMemories()` computes salience scores for each formation memory based on current drive state, then applies configurable rendering strategies.

**Theoretical basis:** Klinger's current concerns theory — active drives increase salience for drive-related memories. Operationalized via PAD proximity (Mehrabian, 1996; Bower, 1981).

**Mechanism:**
1. Read current drive intensities via `resolveAdaptedDrives()`
2. For each drive, compute PAD centroid (seeded from initial config, evolves with reinforcement)
3. For each formation memory, compute association strength to each drive via PAD similarity
4. Overall salience = max association strength across all drives, weighted by drive intensity
5. Apply rendering strategies based on salience scores

**PAD similarity function:** Weighted dot product with valence emphasis (values normalized to [-1, 1] range):
```
similarity = w_p × (mem.pleasure × drive.pleasure)
           + w_a × (mem.arousal × drive.arousal)
           + w_d × (mem.dominance × drive.dominance)
```
Default weights: `w_p = 0.5, w_a = 0.25, w_d = 0.25` (valence most predictive per literature).

**Cold-start strategy:** Initial drive PAD profiles specified in social-config.yaml:
```yaml
drives:
  - type: protection
    intensity: 0.8
    pad: { pleasure: 0.6, arousal: 0.5, dominance: 0.6 }
    triggers: ["threat", "danger", "protect", "guard", "safety"]
```
Initial PAD seeds the centroid. As reinforcement history accumulates, the centroid blends initial values with experienced PAD. Initial weight decays via `manor.drives.initial-pad-decay` config.

**Three rendering strategies (all tunable at runtime):**

1. **Prominence ordering** (`manor.salience.ordering`, default true): Memories ordered by salience score (highest first). All memories still render. Attention guided, no information lost.

2. **Threshold filtering** (`manor.salience.threshold`, default 0.0 = disabled): Memories below threshold don't render. Reduces context size. Risk: may drop relevant memories.

3. **Enrichment injection** (`manor.salience.enrichment`, default false): High-salience memories get contextual framing: "This memory feels vivid right now" or similar. Most explicit LLM guidance. Risk: echo-scripting if not carefully worded.

Strategies compose — ordering + threshold + enrichment can all be active simultaneously. Optimal blend discovered through micro experiments.

**Key files:**
- `wacky-manor/.../CharacterCognition.java` — `renderFormationMemories()` gains salience computation
- `wacky-manor/.../SocialConfig.java` — `Drive` record gains `pad` and `triggers` fields
- `wacky-manor/.../social-config.yaml` — add `pad` and `triggers` to drive entries

### 3.5 Rendering Cleanup: Remove Somatic Drive Text

**What changes:** `renderInternalState()` removes somatic drive descriptions, rendering ONLY CAPS attractors in "What Pulls at You".

**Current:** Renders both somatic drive text (filtered by intensity > 0.2) AND CAPS attractors (filtered by strength > 0.1).

**After:** Renders CAPS attractors only. Drives influence behavior through computation (integrations 1-3), not through prompt text.

**Why remove somatic text:**
- Somatic cues risk echo-scripting (LLM repeats "tension in my hands")
- Somatic cues overlap with personality facets (both describe the same emotional landscape)
- Somatic cues alone score 3/5 = baseline (no additive signal per eval data)
- CAPS attractors are genuinely emergent — behavioral patterns from computation

**CRITICAL PHASING:** This change MUST NOT happen until integrations 1-3 are wired AND verified via micro experiments. D7's strict phasing order prevents the #115 regression.

### 3.6 Content: Enrich Formation Memories

**What changes:** Formation memories for all 5 characters with drives (PP, HC, PP, DD, Mob) enriched to the format proven by eval tests to score 5/5.

**Enrichment adds WITHOUT naming drives:**
1. Child's behavioral RESPONSE to each moment
2. Emotional PAYOFF in capability/motivation terms
3. Additional episodes reinforcing motivational patterns

**Example (PP, age 5):**
> Before: "Your father took you to watch a race. Afterward he knelt down and said: 'A man is measured by what he protects, not what he wins.' You didn't fully understand, but you remembered."
>
> After: "Your father took you to watch a race. Afterward he knelt down and said: 'A man is measured by what he protects, not what he wins.' You didn't fully understand, but you remembered. You spent the rest of the afternoon pretending to fix things around the house, showing him each one: 'Look, Dad, I fixed it.'"

Apply to both BASELINE and GENERIC profiles in social-config.yaml and social-config-generic.yaml.

## 4. Why This Will Be Different (Addressing the #115 Failure)

Issue #115 showed that removing prescriptive text caused drive expression to collapse. The question is: why will computational integration succeed where simple text removal failed?

**What happened last time:** Somatic drive descriptions ("a warmth moves through you when courtesy matters") were the ONLY mechanism through which drives influenced the LLM. Removing them removed all drive influence. There was nothing computational to replace them because the computational connections prescribed by the underlying theories were never wired:

| Theory | What it prescribes | What was built | What was missing |
|--------|-------------------|----------------|-----------------|
| Scherer (2001) | Motivational state feeds relevance check | SEC checks existed | Character drives never connected (only SDT drives) |
| Mischel & Shoda (1995) | Goals participate as CAPS activations | CAPS engine existed | Drive intensities never injected into settling |
| Klinger (1971) | Active drives increase salience for related stimuli | Formation memories existed | No salience modulation |

The architecture had the infrastructure but not the connections. Removing prescriptive text exposed the missing connections — there was no computational fallback.

**What's different now:** We're wiring the three connections the theories prescribe. Each replaces a specific function that prescriptive text previously served:

| What prescriptive text did | What computation now does |
|---|---|
| Told LLM to feel gallant ("warmth when courtesy matters") | Drive→CAPS: gallantry activation biases network toward prosocial attractors → LLM sees emergent attractor ("Something deep-rooted — a tendency-to-protect") |
| Made threats feel important via somatic cues | Drive→Appraisal: protection drive amplifies relevance for threat events → LLM sees stronger emotional responses |
| All backstory equally present regardless of current state | Drive→Memory: high-protection drive makes protection-related formation memories more salient → recent experience shapes which past echoes |

**Evidence from the emergence literature:**
- Park et al. (2023): LLM agent behavior emerges when architectural support (memory, reflection, planning) is present. Each component critically contributes — ablation confirms.
- SEAA (2026): Agents develop distinct behavioral signatures through computational mechanisms (HMM inertia, social comparison), not through explicit instructions.
- Failure mode documented: "LLMs struggle to integrate nuanced events unless explicitly instructed" — BUT this applies when there is no computational pipeline. Our pipeline provides the computational equivalent of "explicit instruction" without prescriptive text.

**The phasing gate makes regression impossible:** Phase 5 (remove somatic text) has a hard prerequisite: phases 1-3 must be wired AND verified. The verification is empirical — micro experiments must show measurable drive expression through the computational pipeline BEFORE prescriptive text is removed. If computation doesn't produce sufficient drive expression, we don't remove the text.

## 5. Implementation Phasing (Strict Order)

| Phase | What | Validates | Gate |
|-------|------|-----------|------|
| 1 | Wire D1 (drive→CAPS activations) | CAPS attractors change when drives adapt | Micro experiment: compare attractors before/after 50+ events |
| 2 | Wire D3 (drive→appraisal) | Emotional responses differentiate by drive profile | Micro experiment: same event, different drive profiles → different appraisal intensity |
| 3 | Wire D5 (drive→memory salience) | Memory ordering changes with drive state | Micro experiment: vary drive intensity, observe memory salience scores |
| 4 | Enrich formation memories | Enriched memories score ≥4/5 in eval | Run RelationalModelEvalTest |
| 5 | Remove somatic drive text | No regression — drive expression maintained | Run full drive expression eval, compare to pre-removal baseline |

Each phase has a verification gate. Phase 5 (rendering cleanup) MUST NOT proceed until phases 1-3 are verified. This prevents the #115 regression.

## 5. Configuration Summary

| Config Property | Default | Purpose |
|-----------------|---------|---------|
| `manor.caps.drive-dampening` | 0.3 | Prevents drive activations from overwhelming situations |
| `manor.caps.drive-activation-enabled` | true | Master toggle for drive→CAPS |
| `manor.appraisal.character-drive-weight` | 1.0 | Scaling for character drive relevance |
| `manor.appraisal.character-drives-enabled` | true | Master toggle for drive→appraisal |
| `manor.salience.ordering` | true | Order memories by salience |
| `manor.salience.threshold` | 0.0 | Filter memories below threshold (0 = disabled) |
| `manor.salience.enrichment` | false | Add contextual framing to salient memories |
| `manor.drives.initial-pad-decay` | 0.95 | Decay rate for initial PAD seeds vs experienced PAD |
| `manor.salience.valence-weight` | 0.5 | Valence weight in PAD similarity |
| `manor.salience.arousal-weight` | 0.25 | Arousal weight in PAD similarity |
| `manor.salience.dominance-weight` | 0.25 | Dominance weight in PAD similarity |

All parameters tunable at runtime per D4a. Optimal values discovered empirically through micro experiments.

## 6. Emergence Constraints

All changes must satisfy:

- **No raw values exposed** — LLM never sees drive type labels, intensity numbers, or cognitive model internals
- **No prescriptive text** — no somatic cues, behavioral directives, or "you should feel X" prompting (removed in phase 5)
- **No triggering** — no scenario-specific cues that make the LLM seek matching situations
- **No register bias** — different characters express drives through different behavioral modes
- **Adaptation must be testable** — before/after drive adaptation must produce measurably different behavioral output
- **Literature-grounded** — every mapping, threshold, and model value traceable to published research per D2 sourcing tiers

## 7. Cross-Repo Impact

| Repo | Changes |
|------|---------|
| `neocortex/mindmap-intelligence` | BehavioralSynthesisPhase: read drive-intensity nodes, inject as activations |
| `neocortex/cognition` | AppraisalTickParticipant: merge character drives, pass to SEC checks |
| `neocortex/cognition-api` | Drive record: extend trigger to List<String> |
| `examples/wacky-manor` | CharacterCognition: salience rendering, somatic removal. SocialConfig: pad/triggers fields. social-config.yaml: enriched memories + drive config |
| New config | drive-caps-mapping.yaml: universal drive-type → CAPS activation mapping |

## References

### Primary Theory
- Mischel, W. & Shoda, Y. (1995). A cognitive-affective system theory of personality. *Psychological Review*, 102(2), 246-268.
- Read, S.J. & Miller, L.C. (2002). Virtual Personalities: A neural network model of personality. *Personality and Social Psychology Review*, 6(4), 357-369.
- Read, S.J., Brown, A.D., Wang, P. & Miller, L.C. (2018). The neurobiological basis of personality. *Personality Neuroscience*, 1, e10.
- Scherer, K.R. (2001). Appraisal Considered as a Process of Multilevel Sequential Checking. In Scherer, Schorr & Johnstone (Eds.), *Appraisal Processes in Emotion*. Oxford UP.
- Lazarus, R.S. (1991). *Emotion and Adaptation*. Oxford UP.
- Klinger, E. & Cox, W.M. (2004). Motivation and the Theory of Current Concerns. In *Handbook of Motivational Counseling*.
- Mehrabian, A. (1996). Pleasure-arousal-dominance: A general framework for describing and measuring individual differences in Temperament. *Current Psychology*, 14, 261-292.

### Supporting Evidence
- Bower, G.H. (1981). Mood and memory. *American Psychologist*, 36(2), 129-148.
- Rubin, D.C. & Talarico, J.M. (2009). A comparison of dimensional models of emotion. *Memory*, 17(8), 802-808.
- Moors, A., Ellsworth, P.C., Scherer, K.R. & Frijda, N.H. (2013). Appraisal Theories of Emotion: State of the Art. *Emotion Review*, 5(2), 119-124.
- Smith, C.A. & Kirby, L.D. (2009). Putting appraisal in context. *Cognition and Emotion*, 23(7), 1352-1372.
- Sander, D., Grandjean, D. & Scherer, K.R. (2005). A systems approach to appraisal mechanisms in emotion. *Neural Networks*, 18, 317-352.
- Park, J.S. et al. (2023). Generative Agents: Interactive Simulacra of Human Behavior. *UIST 2023*.
- Bradley, M.M. & Lang, P.J. (1999). Affective Norms for English Words (ANEW). Technical Report C-1, University of Florida.
- Warriner, A.B. et al. (2013). Norms of valence, arousal, and dominance for 13,915 English lemmas. *Behavior Research Methods*, 45, 1191-1207.

### Drive-Specific Mapping Sources
- Batson, C.D. (1991). *The Altruism Question*. Lawrence Erlbaum. (prosocial/gallantry)
- Penner, L.A., Dovidio, J.F., Piliavin, J.A. & Schroeder, D.A. (2005). Prosocial Behavior. *Annual Review of Psychology*, 56, 365-392. (prosocial/gallantry)
- McClelland, D.C. (1961). *The Achieving Society*. Van Nostrand. (achievement/proving-worth)
- George, C. & Solomon, J. (2008). The Caregiving System. In Cassidy & Shaver (Eds.), *Handbook of Attachment*. (protection/caregiving)
- Christie, R. & Geis, F.L. (1970). *Studies in Machiavellianism*. Academic Press. (scheming)

### Decisions
- `wsp-casehub/specs/issue-115-drive-expression-gap/decisions.md` (D1-D8)
- Issue #115 regression data (gloating -1.52 collapse)
- Issue #118 root-cause analysis
- Issue #119 Klinger's current concerns
