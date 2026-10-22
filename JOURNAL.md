# Design Journal — issue-122-drive-expression-pipeline

## 2026-10-22: Multi-channel thinking prompt — literature grounding

### Problem

The `CharacterAgentLoop` thinking prompt directed characters to start "in the body, not the head" and listed only somatic sensations (tightness, warmth, prickle, numbness). This created a bottleneck: the neocortex `DriveDescriptionGenerationPhase` had already moved to four-channel expression, but the runtime prompt still funnelled character cognition through a single somatic channel.

The somatic-only constraint was already documented as harmful — `DriveDescriptionDiscriminationTest` showed discrimination scores dropping from 3.97 to 2.16 when the generation prompt restricted to somatic-only, because different drives produce similar body sensations (warmth can be rage, care, or lust).

### Literature basis for multi-channel expression

The four channels are not arbitrary — each is grounded in distinct empirical traditions:

**1. Somatic** — bodily sensations (warmth, tension, weight, tingling)
- Damasio, A. (1994). *Descartes' Error: Emotion, Reason, and the Human Brain.* Putnam.
- Barrett, L.F. (2017). The theory of constructed emotion. *Social Cognitive and Affective Neuroscience*, 12(1), 1–23.
- Dunn, B.D., Dalgleish, T., & Lawrence, A.D. (2006). The somatic marker hypothesis: A critical evaluation. *Neuroscience & Biobehavioral Reviews*, 30(2), 239–271.

Dunn et al. found patients showing impaired decision-making *without* somatic marker deficits, and vice versa — somatic markers are **neither necessary nor sufficient** alone. This is the empirical case against somatic-only.

**2. Attentional** — what captures attention, where awareness is drawn
- Easterbrook, J.A. (1959). The effect of emotion on cue utilization and the organization of behavior. *Psychological Review*, 66(3), 183–201.

Easterbrook's cue-utilization theory: emotional arousal narrows the attentional field. Different drives narrow attention to different targets — scheming scans for leverage, protection scans for threats, curiosity scans for novelty. This channel is inherently discriminating.

**3. Temporal** — how the moment feels (suspended, urgent, stretching, compressed)
- Barrett, L.F. (2017). The theory of constructed emotion. *SCAN*, 12(1), 1–23.

Barrett's predictive coding account: the brain constructs temporal experience as part of emotion. Fear compresses time (urgency), care stretches it (patience), play suspends it (flow). Temporal distortion is a discriminating signal that somatic sensation alone cannot capture.

**4. Action tendency** — what the person feels pulled toward doing
- Frijda, N.H. (1986). *The Emotions.* Cambridge University Press.

Frijda's core claim: emotions ARE action tendencies. Not consequences of emotion — constitutive of it. Each drive produces a characteristic pull: protection pulls toward shielding, scheming pulls toward manoeuvring, curiosity pulls toward investigating. This is the most discriminating channel.

### Why Scherer's CPM is the organising framework

Scherer, K.R. (2001). Appraisal considered as a process of multi-level sequential checking. In *Appraisal Processes in Emotion* (pp. 92–120). Oxford University Press.

Scherer's Component Process Model defines five synchronised subsystems: cognitive appraisal, neurophysiological support (somatic), motivational executive (action tendency), motor expression, and subjective feeling. Our four channels map to four of Scherer's five — we omit motor expression because the characters express through dialogue and action, not facial/gestural channels.

### What changed

Single commit `f01446d`: replaced the somatic-only thinking prompt with a four-channel version that walks the character through somatic → attentional → temporal → action tendency, in the same order as the neocortex generation phase. The prompt stays intimate and non-technical — it's guidance for a character, not a taxonomy.

### What we verified

- **Formation memories complete:** all 14 unique drive types (17 entries across 5 characters) have serving formation memories with strength annotations. No gaps.
- **631 fast tests pass** — no regressions from the prompt change.
- **Eval tests unchanged** — `SomaticDescriptionEvalTest`, `FocusedDriveEvalTest`, and `PrioritySignatureReproducerTest` use standalone prompts for somatic-only baseline comparison. Historical documentation, not runtime prompts.

### What remains (neocortex-scoped)

- `DriveProfileDerivationPhase` not producing profiles in full scenario runs
- Somatic discrimination text not generating (depends on above)
- Situation-dependent channel selection via vocabulary classifier

### Key principle

**P-channel: Somatic markers are necessary but not sufficient for drive discrimination.** The empirical evidence (Dunn et al. 2006, our own discrimination eval 3.97→2.16) shows that somatic-only expression collapses drive distinctiveness. Multi-channel expression — somatic + attentional + temporal + action tendency — provides the discriminating dimensions that somatic alone cannot.
