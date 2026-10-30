# Design Journal — wip/era5-relational-eval

## 2026-10-29 — Pipeline re-architecture for targeted somatic drives (#175)

### The problem

Era 5 added relational schemas (#477) with five layers (types/storage, appraisal wiring, two-channel prompt, dynamic evolution, CAPS biasing). All working mechanically. But adding the relational context to the character prompt **degraded** somatic register from 3.74 to 3.13. Quality held at ~3.84, so the prose was good — just in the wrong register.

The biggest drops were all on inherently social drives: DD gloating (-1.54), DD recognition (-2.13), PP gallantry (-1.25), AHM loyalty (-0.61). Non-social drives (curiosity, scheming) held steady.

### Three-layer diagnosis

**Layer 1 — Redundancy.** Relational information about the same characters appeared in 4+ separate sections: Social Awareness, Theory of Mind, "How You Feel About Those Present," "How You Bond," and Trust sections.

**Layer 2 — Instruction competition.** The Barrett somatic instruction says "feel, don't think." The relational sections say "Prey. Trust 0.1. Predictable." — cognitive-analytical framing. The LLM has two registers available and gravitates to the easier one (cognitive). This is why register drops while quality holds.

**Layer 3 — Phenomenological category error.** Drives and relationships are rendered as independent dimensions in separate sections. But social drives ARE their targets — you gloat AT someone, protect SOMEONE. Separating them forces the LLM to cross-reference ("Reginald is prey" in section 11 + "gloating is active" in section 8) under attention pressure from 14 sections. That bridge fails.

### Critical correction: don't remove cognitive abilities

Initial approach was to remove the relational/attachment/trust sections. **Wrong.** Those sections represent cognitive abilities — the character's capacity to reason about relationships, attachment, and trust. Removing them lobotomizes the character to fix a rendering problem.

The fix is not less cognition — it's better wiring: correct appraisal (drives know their targets) and targeted prose (somatic descriptions reference who they're about).

### Architecture decisions

1. **Pipeline:** `DriveDescriptionGenerationPhase` (P22) now reads relational schemas (available from `RelationalSchemaEvolutionPhase` at P20) and generates per-drive-per-role Barrett descriptions. One LLM call per drive with all schemas as context. Pattern follows `OtherModelDerivationPhase` (P25).

2. **Appraisal:** `CognitionObservationBuilder.resolveRelationalTargets()` — scene-level target resolution after `appraisalFilterDrives()`. Not in `AppraisalTickParticipant` (wrong scope — per-tick, single target) or CAPS (wrong abstraction — situation->behavior, not relational targeting).

3. **Composition:** Drives render as `GLOATING (0.7) -> Penelope: [targeted somatic description]`. Falls back to untargeted description when no role-specific match exists. All cognitive sections preserved.

4. **Drive-role salience:** `computeDriveRoleSalience()` maps drive types to compatible schema roles (gloating->prey/rival, gallantry->beloved/ward, etc.). This is the semantic bridge — lives in neocortex, not examples.

5. **Evolved schema rendering:** `renderRelationalSchemas()` now reads from MindMapStore overlay nodes (evolved data), falling back to YAML when no overlays exist.

### Key finding: DriveDescriptionGenerationPhase had zero relational context

The somatic descriptions generated during consolidation received personality facets, attachment style, and formation memories — but zero relational schemas. Descriptions were target-agnostic. This was the deepest gap — the pipeline computed relationships at P20 but never fed them to description generation at P22.

### What was NOT done

- `computeDriveRoleSalience` uses a hardcoded switch statement. Future: consider making this data-driven.
- Theory of Mind sections not touched. If register doesn't fully recover, they're the next candidate for review.
- No eval run yet — needs consolidation cache clear + scenario run.

---

## 2026-10-30 — Deep register diagnostic + golden prose methodology (#175)

### The diagnostic journey

Started with the 30-event eval: register at 3.46 (up from 3.13, but still below Era 4's 3.74). The investigation went through multiple layers of "but why":

**Layer 1: "Era 5 added analytical sections"** — initial hypothesis. Checked the code. **Wrong.** All 15 section slots existed in Era 4 too. The section structure is identical. Diff between Era 4 and Era 5 CharacterCognition shows only two changes: 4-arg→5-arg renderInternalState call, and evolved overlay reading.

**Layer 2: "The neocortex modules changed what goes into the sections"** — partially right. The #716-#720 commits added RelationalBiasing, OtherModel, SchemaEvolution. These changed the content flowing through sections that were already present.

**Layer 3: "The 'What Pulls at You' section grows from 9→25 parts"** — verified. After consolidation, the section adds 4 self-standards, 6 differentiation texts, 6 priority signatures. But then discovered these are ALSO somatic (not analytical as initially assumed). "How they differ: One pull settles the body... the other sharpens..." is Barrett-quality channel differentiation. So volume alone isn't the problem.

**Layer 4: "Action readiness channel drops to zero"** — the quantitative smoking gun. Era 4 had action readiness markers in every aside (avg 1.9). Era 5 had zero. But the generated drive descriptions CONTAIN action readiness vocabulary (3-4 markers each). The descriptions have the palette; the LLM just doesn't use it.

**Layer 5: "Analytical labels short-circuit action readiness"** — the root cause. Trust ("LOW"), Social Awareness ("more cooperative"), relational schemas ("prey") give the LLM ready-made labels for classifying other people. In Era 4, the LLM had to CONSTRUCT its understanding of others through body-based action readiness ("my weight shifts when he moves"). In Era 5, it just CLASSIFIES ("he's the bookish type") because the labels are easier.

**Layer 6: "But those labels were in Era 4 too"** — the relational schemas from YAML were always rendered. So the labels aren't new. What changed is how MUCH analytical content is present (relational biasing, evolved schemas, OtherModel) and how it interacts with the consolidated cognitive content (25-part section).

**Layer 7: "Era 4 also degrades over time"** — verified. Era 4 300-tick run: early asides body=2.6 attn=2.2 action=1.9. Late asides: body=1.4 attn=0.4 action=1.0. The register degradation is not Era-5-specific. Consolidation-added content dilutes all register over time in BOTH eras. Era 5 just starts from a worse position.

### The Barrett 4-channel correction

Mid-diagnostic, the user pointed out I was conflating "somatic" (body channel only) with all four Barrett channels. The framework has four EQUAL channels: body, attention, time, action readiness. My entire analysis was body-biased. This reframing was critical — it revealed that action readiness is the most vulnerable channel because it's most easily replaced by analytical planning language.

The user also identified that Era 4's channel selection was DYNAMIC — not all four every time, but the 1-2 most relevant to the drive×target×situation. This made Era 4 feel alive rather than formulaic.

### The golden prose methodology

The biggest methodological shift in this session: **create the ideal rendered output FIRST, test it scores well, THEN make the pipeline generate it.**

Previous approach: change the pipeline → run scenario → evaluate → discover what works → iterate. Slow, expensive (each run is 30-50 min + classifier time), and the search space is enormous.

New approach: hand-craft the ideal thinking/aside text → run classifier (seconds, not minutes) → iterate on prose until all drives hit 4.0+ → THEN implement the backend to generate equivalent content. Much faster iteration, clear target, measurable convergence.

The user generalized this: "each new facet must first describe its contribution to rendered output, we then test that. if it works, we then implement the backend."

### Agent orchestration — what we learned

Parallel forks per character worked well for initial prose creation. Five agents, each writing 10 events, running the classifier, iterating. All completed in ~5 minutes with strong results.

**Targeted patch agents** for filling cognitive layer gaps (attachment, beliefs, ToM) worked when the additions were WOVEN IN, not bolted on. A review agent caught two issues (metaphor collision between characters, voice register drift) that individual agents couldn't see.

**The deep analysis agent** (effort:max, non-fork) produced the best single improvements: DD Q=4.07→4.36, R=3.88→4.36, D=3.63→4.10. It discovered the body-location differentiation principle and the temporal ordering principle. Worth the 30-minute runtime.

**Over-trigger reduction agents** revealed a fundamental limit: some drives are IDENTITY-LEVEL. You can't subtract scheming from hooded-claw's voice without destroying the character. Three agents independently discovered this limit. The insight: for identity-level drives, improvement requires ADDITION (new events with genuinely different content) or REFRAMING (sub-types, per-target body-location variation), not subtraction.

### The remaining puzzle: villain scheming differentiation

DD scheming Q=3.62, R=3.88, D=3.38 is the floor of the entire corpus. HC scheming R=3.89, HC self-preservation Q=3.80, HC gloating D=3.75 are also below 4.0. All four are intertwined — scheming vocabulary triggers scheming detection even in events intended as pure gloating.

**Why this is hard:** Scheming's natural channel is ATTENTION (mapping, calculating, tracking). Attention-channel language is inherently cognitive-adjacent ("noticed", "mapped", "calculated"). The classifier may read attention-channel scheming as lower register because the vocabulary overlaps with analytical narration. Meanwhile, body-channel drives (gloating → warmth, protection → weight-dropping) have vocabulary that CANNOT be confused with analysis.

**Possible approaches for next session:**
- Scheming sub-types (defensive vs offensive) with different channel selections
- Genuinely non-scheming events (childhood memory, aesthetic response, physical discomfort)
- Per-target body-location variation for scheming
- Temporal ordering to separate scheming from gloating ("the plan was done... THEN the warmth arrived")
- Classifier investigation for scheming-detection bias

### Key architectural decisions

All locked in, ready for pipeline implementation once golden prose is complete:
- Drive-first rendering with per-role channel-selected modulations
- Mechanical channel selection via lookup tables (no LLM for selection)
- Relational information folds INTO drives, not alongside
- seedPeople() must write relational schema properties into overlay nodes
- Describe → test → implement methodology for ALL cognitive extensions
