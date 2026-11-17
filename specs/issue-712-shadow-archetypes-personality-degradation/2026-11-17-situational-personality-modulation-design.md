# Situational Personality Modulation — Design Spec

Ref: casehubio/neocortex#712

## Goal

Add situation-dependent personality modulation to the cognitive pipeline. Characters' PID-5 personality facets shift in response to detected situational states (stress, flow, social context), producing personality-consistent behavioral changes through the existing CDE derivation pipeline. Prolonged or extreme exposure causes permanent personality drift — the elastic band model.

## Background

### What exists

The personality pipeline (completed in #845) flows: Engram → Eidos `AgentDisposition` → `EidosCognitiveProfileBridge` → `CognitiveDerivationEngine` → `CognitiveDefaults`. The CDE has 14+ derivation pathways that map PID-5 facets to cognitive parameters (appraisal weights, regulation capacities, search budgets, expression styles, etc.).

CAPS settles behavioral attractors from situational input activations through a network of mediating and output nodes across 7 categories (approach_avoidance, social, conflict, achievement, autonomy, learning, regulation, coping). Stress-related behavior is emergent — threat inputs activate conflict outputs (fight/flight/freeze/fawn) and regulation/coping outputs.

`PersonalityEvolutionOrchestrator` manages long-term personality change via trait pressure on Jungian function weights. `DispositionSignalStore` accumulates activation signals with decay.

### What's missing

No mechanism detects "this character is under stress" and modulates their personality accordingly. CAPS produces stress-related attractors but nothing interprets them as personality-shifting events. The pipeline produces the same cognitive defaults regardless of situational state.

### Core model: elastic band

The personality state is a single continuously-evolving vector. Situational context changes what the personality is pulled toward — not a separate overlay, but a modulation of the same evolution process.

Like an elastic band: situations stretch the personality away from its resting state. Brief/mild stretches return nearly to the original. Prolonged, extreme, or repeated stretching permanently deforms the band — the resting state itself shifts.

Three properties:
1. **Immediate discrete pull** — entering a situation shifts effective personality
2. **Continuous drift** — ongoing exposure increases the shift
3. **Imperfect return** — brief returns nearly to home; prolonged/extreme permanently deforms

## Architecture

### Class/instance model

Eidos is the **class definition** — the immutable personality template including situational attractor profiles. The runtime personality state is an **instance** — starts as a copy, evolves through experience. The CDE reads the instance, not the class.

| Concept | Store | Mutability |
|---|---|---|
| Situational attractor profiles (facet deltas per archetype × mode) | Eidos topology YAML (class) | Immutable |
| Base personality (home attractor) | Runtime instance | Slow drift |
| Current personality (elastic position) | Runtime instance | Per-tick |
| Elasticity per facet | Runtime instance | Degrades with exposure, recovers during rest |
| Effective personality (CDE input) | Computed | Stateless |

### Elastic band state

Three values per PID-5 facet (13 facets × 3 = 39 values per agent):

- **`current`** — where the personality IS right now. Moves toward the active attractor each tick.
- **`home`** — where the personality gravitates when no situation is active. Drifts slowly toward `current` (permanent deformation).
- **`elasticity`** — how well the personality returns to home (0.0 = fully deformed, 1.0 = perfectly elastic).

**Initialization:** At character creation, `current = home = Eidos baseline facet scores`. `elasticity = 1.0` for all facets.

**Per-tick update (situational pull active):**
```
target = home + attractor_delta × intensity
current = current + pull_rate × (target - current)
home = home + home_drift_rate × (1 - elasticity) × (current - home)
elasticity = max(elasticity_floor, elasticity - degradation_rate × intensity)
```

**Per-tick update (no situation active — rest):**
```
current = current + return_rate × elasticity × (home - current)
elasticity = min(1.0, elasticity + recovery_rate)
```

**Elasticity floor (metal fatigue):** Each deformation episode lowers the floor by a small increment. Repeated stress cycles permanently reduce the character's ability to return fully. The floor itself never recovers — it represents structural damage to personality resilience.

```
elasticity_floor = max(0.0, elasticity_floor - fatigue_increment × intensity)
```

**Elasticity is per-facet, not per-mode-per-facet.** A facet stretched by stress is also less elastic when pulled by flow or social context. Deformation is structural — the personality's resilience on that dimension degrades regardless of which situation caused it.

### Situational modes

Three modes, each with detection criteria and attractor profiles:

| Mode | Trigger signals | Psychological meaning |
|---|---|---|
| **Stress** | Conflict attractors (fight/flight/freeze/fawn), low mood pleasure, high arousal, threat appraisal | Character under threat, loss, or resource scarcity |
| **Flow** | Achievement/learning attractors (persist, explore, consolidate), positive mood, moderate arousal, competence drive | Character at peak functioning, fully engaged |
| **Social** | Social attractors (trust/distrust/proximity patterns), affiliation drive, relational appraisal | Character's personality shifts based on relational dynamics |

Each mode is detected independently and can be active simultaneously. A character can be stressed AND in a competitive social context — the attractor deltas compose additively.

### Detection: weighted threshold classifier

Each mode's intensity is a weighted sum of signals:

```yaml
modes:
  stress:
    detection:
      signals:
        - source: caps_attractor
          nodes: [fight, flight, freeze, fawn]
          aggregation: max
          weight: 0.4
        - source: mood
          dimension: pleasure
          invert: true  # low pleasure = high stress signal
          weight: 0.25
        - source: mood
          dimension: arousal
          weight: 0.2
        - source: appraisal
          dimension: threat
          weight: 0.15
      threshold: 0.45
      fade_rate: 0.1  # intensity decays at this rate when signals drop
```

The detector runs every tick and produces a `SituationalState` — a map of `{mode → intensity}` for all modes above threshold.

**Fade dynamics:** Mode intensity doesn't snap to zero when signals drop. It fades at `fade_rate` per tick, matching psychological reality — you don't snap out of stress instantly. This prevents oscillation at the threshold boundary.

### Situational attractor profiles (topology)

Defined per archetype family in a YAML topology file, grounded in Jungian shadow theory and archetype psychology literature. 12 families × 3 modes = 36 profiles.

```yaml
archetype_profiles:
  CREATOR:
    stress:
      description: "Obsessive self-validation, perfectionism paralysis"
      facet_deltas:
        Anxiousness: +0.15
        Emotional Lability: +0.20
        Grandiosity: +0.10
        Withdrawal: +0.10
    flow:
      description: "Expansive creative expression, boundary dissolution"
      facet_deltas:
        Risk Taking: +0.15
        Impulsivity: +0.10
        Grandiosity: +0.05
        Anxiousness: -0.10
    social_competitive:
      description: "Defensive posturing, dismissive of others' work"
      facet_deltas:
        Hostility: +0.10
        Grandiosity: +0.15
        Suspiciousness: +0.10

  CAREGIVER:
    stress:
      description: "Martyrdom, boundary collapse, resentful compliance"
      facet_deltas:
        Emotional Lability: +0.20
        Anxiousness: +0.15
        Callousness: -0.10  # over-empathy, not under
        Withdrawal: -0.15   # unable to withdraw, over-extends
    flow:
      description: "Effortless nurturing, secure presence"
      facet_deltas:
        Anxiousness: -0.15
        Emotional Lability: -0.10
        Suspiciousness: -0.10
    social_competitive:
      description: "Passive-aggressive accommodation"
      facet_deltas:
        Hostility: +0.10
        Manipulativeness: +0.10
        Grandiosity: -0.05

  # ... remaining 10 families
```

Each profile is literature-grounded. The descriptions are for documentation — only `facet_deltas` are consumed by the engine.

**Social mode subtypes:** The social mode has subtypes (competitive, collaborative, hierarchical) rather than a single profile. The subtype is detected from the social attractor pattern — trust+proximity = collaborative, distrust+distance = competitive, comply/submit = hierarchical. Each subtype has its own facet deltas.

### Pipeline integration

**New component:** `SituationalPersonalityParticipant` — a tick participant.

**Ordering:** Runs after `BehavioralSynthesisPhase` (which produces CAPS attractors) and `MoodOrchestrator` (which produces current mood), but before `PersonalityEvolutionOrchestrator` (which processes trait pressure). Priority must be set to enforce this ordering.

**Per-tick flow:**

1. Read multi-signal state: CAPS `SettlingResult` attractors, mood PAD, `DriveProfile`, appraisal weights
2. Classify situational modes via weighted threshold detector → `SituationalState`
3. Look up active attractor profiles for the agent's archetype family
4. Compute composite attractor delta: `Σ(profile_delta × mode_intensity)` across all active modes
5. Update elastic band state: pull `current` toward composite attractor, drift `home`, degrade `elasticity`
6. If `current` changed since last tick (dirty check), build modified `DescriptorView` with `currentFacets` and call `CDE.deriveAndMerge()` via the bridge to refresh `CognitiveDefaults`
7. Persist updated state to `SituationalPersonalityStore`

**Bridge interaction:** The tick participant calls `EidosCognitiveProfileBridge.refreshWithModulatedFacets(agentId, currentFacets)` — a new method that builds a `DescriptorView` with the modulated facet scores and invokes `CDE.deriveAndMerge()`. The bridge owns CDE invocation; the tick participant does not bypass it.

### Persistence

**New store:** `SituationalPersonalityStore` (neocortex-local, SQLite).

Schema:
```sql
CREATE TABLE situational_personality_state (
    agent_id    TEXT NOT NULL,
    tenant_id   TEXT NOT NULL,
    facet       TEXT NOT NULL,
    current_val REAL NOT NULL,
    home_val    REAL NOT NULL,
    elasticity  REAL NOT NULL,
    elasticity_floor REAL NOT NULL,
    updated_at  TEXT NOT NULL,
    PRIMARY KEY (agent_id, tenant_id, facet)
);
```

Co-located with other neocortex stores (alongside `SqliteCapsEngine`). Avoids cross-repo API changes to eidos.

**Load/save pattern:** Load full agent state on first tick access (13 rows per agent), cache in memory, save dirty state at end of tick. Same pattern as `AgentCapsState` loading in `SqliteCapsEngine`.

### Observability

**Cognitive journal entry:** New `CognitiveEntry.SituationalModulation` record:
```
(tenantId, agentId, timestamp, modes: Map<Mode, Double>,
 compositeDelta: Map<Facet, Double>, elasticitySnapshot: Map<Facet, Double>)
```

**Pipeline trace:** `PipelineTrace.SituationalPersonalitySnapshot` captures mode intensities, active profiles, and effective facet values for debugging.

### Integration with PersonalityEvolutionOrchestrator

The elastic band model's `home` drift IS the personality evolution mechanism for situational effects. However, the existing `PersonalityEvolutionOrchestrator` manages Jungian function evolution separately.

These are complementary, not competing:
- **Jungian evolution:** Trait pressure on cognitive function weights (Ti, Te, Fi, Fe...) from relational events. Long time scale.
- **Situational modulation:** PID-5 facet elasticity/drift from detected situational states. Medium time scale.

Both feed into the effective personality that CDE reads. The bridge merges both channels: Jungian weights from the evolution orchestrator + modulated facet scores from the situational modulator → `DescriptorView` → `CDE.derive()`.

## Parameters

All parameters defined in the topology YAML and tunable via eval:

| Parameter | Default | Description |
|---|---|---|
| `pull_rate` | 0.3 | How fast `current` moves toward the active attractor per tick |
| `return_rate` | 0.2 | How fast `current` returns toward `home` during rest |
| `home_drift_rate` | 0.02 | How fast `home` moves toward `current` (permanent deformation rate) |
| `degradation_rate` | 0.01 | How fast `elasticity` degrades per tick under exposure |
| `recovery_rate` | 0.005 | How fast `elasticity` recovers during rest |
| `fatigue_increment` | 0.002 | How much `elasticity_floor` drops per deformation episode |
| `mode_fade_rate` | 0.1 | How fast mode intensity decays when detection signals drop |
| `mode_threshold` | 0.45 | Minimum weighted signal sum for mode activation |

## Testing

### Unit tests (pure functions)

- **Elastic band mechanics:** Given initial state + attractor delta + intensity, verify current/home/elasticity update correctly. Test: brief exposure returns nearly to home; prolonged exposure deforms permanently; repeated episodes lower elasticity floor.
- **Mode detection:** Given CAPS attractors + mood + drives + appraisal, verify correct mode classification and intensity. Test: each mode individually, simultaneous modes, threshold boundary behavior.
- **Attractor composition:** Given multiple active modes, verify additive delta composition with clamping.
- **Dirty check:** Verify CDE re-derivation only triggers when currentFacets actually changed.

### Integration tests

- **End-to-end pipeline:** Character receives stressful situation input → CAPS settles to conflict attractors → stress mode detected → personality modulated → CDE produces different cognitive defaults → personality returns toward home after stress resolves.
- **Persistence round-trip:** Save elastic state → reload → verify identical state.
- **Evolution integration:** Prolonged stress → elasticity degrades → home drifts → PersonalityEvolutionOrchestrator reads evolved facets.

### Eval validation

- **Behavioral coherence:** Characters under stress should exhibit archetype-consistent degradation (Creator → obsessive self-validation, Caregiver → martyrdom). Score via eval judges.
- **Differentiation:** Different archetypes under the same stress should degrade differently. Compare pre/post stress cognitive defaults across archetypes.
- **Recovery:** Characters should recover personality after brief stress. Prolonged stress should leave lasting traces. Measure facet distance from baseline after recovery.

## Ordering

1. **Topology authoring** — Define situational attractor profiles for 12 archetype families × 3 modes in YAML. Literature research required.
2. **Elastic band engine** — Pure-function state machine: `update(state, attractorDelta, intensity) → newState`. No dependencies.
3. **Mode detector** — Weighted threshold classifier: `detect(capsResult, mood, drives, appraisal) → SituationalState`. No dependencies.
4. **Persistence store** — `SituationalPersonalityStore` SQLite implementation following `SqliteCapsEngine` pattern.
5. **Tick participant** — `SituationalPersonalityParticipant` wiring detector + engine + store + bridge. Depends on 2-4.
6. **Bridge extension** — `refreshWithModulatedFacets()` method on `EidosCognitiveProfileBridge`. Depends on 5.
7. **Observability** — Cognitive journal entry + pipeline trace snapshot. Depends on 5.
8. **Eval run** — Validate behavioral coherence and differentiation. Depends on all.

Steps 2-4 are independent and can be built in parallel. Steps 1 requires literature research and can proceed concurrently with 2-4.

## References

- casehubio/neocortex#712 — Shadow archetypes: situation-dependent personality degradation profiles
- casehubio/neocortex#845 — CDE PID-5 facet upgrade (pipeline unification, AmpdFacet in Eidos)
- `docs/specs/issue-845-cde-pid5-facets/2026-11-13-cde-pid5-facet-upgrade-design.md` — #845 design spec
- `neocortex/docs/research/era7-personality-trait-integration-research.md` — PID-5×Jungian integration research
- `caps-api/.../CapsAttractor.java` — Attractor model
- `caps-engine/.../CapsSettler.java` — Settling algorithm
- `cognition/.../PersonalityEvolutionOrchestrator.java` — Trait pressure evolution
- `cognitive-eidos/.../EidosCognitiveProfileBridge.java` — Eidos→CDE bridge
- `cognitive-index/.../CognitiveDerivationEngine.java` — CDE derivation engine
- `eidos/vocab/.../ArchetypeFamily.java` — 12 archetype families
- Mischel, W. & Shoda, Y. (1995). A cognitive-affective system theory of personality. *Psychological Review*, 102(2), 246-268. — CAPS theory foundation
- Hartwell, M. & Chen, J. (2012). *Archetypes in Branding*. — 12 archetype families with 48 sub-archetypes
- Jung, C.G. (1959). *The Archetypes and the Collective Unconscious*. CW 9i. — Shadow archetype theory
- Krueger, R.F. et al. (2012). PID-5 facet model. — PID-5 foundation
- Gore, W.L. & Widiger, T.A. (2013). PID-5 ↔ FFM mapping. — Big Five bridge
