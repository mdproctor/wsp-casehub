# Session Notes — #122 Drive Expression Pipeline

## What was accomplished (2026-10-22)

### Examples-side work complete

1. **Thinking prompt**: replaced somatic-only with four-channel (somatic, attentional, temporal, action tendency). Commit `f01446d`.
2. **Formation memories**: verified all 17 drive entries (14 unique types) have serving formation memories. No gaps.
3. **Smoke test**: 128-event GENERIC profile run confirmed all 5 characters use multiple channels. Attentional narrowing is the strongest discriminator. Transcript at `wacky-manor/docs/eval/multi-channel-smoke-2026-10-22/`.

### Literature grounding

Four channels grounded in published literature:
- **Somatic**: Damasio 1994, Barrett 2017; Dunn et al. 2006 showed somatic markers are neither necessary nor sufficient
- **Attentional**: Easterbrook 1959 — cue-utilization theory, drive-specific narrowing
- **Temporal**: Barrett 2017 — predictive coding constructs temporal experience during emotion
- **Action tendency**: Frijda 1986 — emotions ARE action tendencies (constitutive, not consequential)
- **Organising framework**: Scherer 2001 CPM — five subsystems, four map to our channels (omit motor expression)

### Artifacts produced
- Journal: `JOURNAL.md` — full literature review and design rationale
- Blog: `blog/2026-10-22-mdp01-multi-channel-drive-expression.md` — "Why Somatic Markers Aren't Enough"
- Smoke test: `wacky-manor/docs/eval/multi-channel-smoke-2026-10-22/`

### Issues filed
- **neocortex#668**: Situation-dependent channel weighting for drive expression — next work item

## Key observations from smoke test

- Attentional narrowing is inherently discriminating — each character's attention targets differ by drive
- Action tendency produces unique signatures per drive ("staying close" = protection, "inventorying" = greed)
- Temporal channel weakest — absent in calm states (Penelope, Peter), present in high-arousal (scheming, protection). Consistent with Barrett
- The prompt change alone — without any code changes — produced multi-channel thinking across all characters

## Next session: neocortex#668

Situation-dependent channel weighting. The four channels currently have equal weight. Easterbrook's work predicts attentional narrowing varies with arousal. Barrett's model predicts temporal distortion varies with situation. A vocabulary classifier should weight channels by situation context.

Related neocortex issues: #607 (Frijda action readiness), #587 (graduated SituationClassifier).
