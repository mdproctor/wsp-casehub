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
