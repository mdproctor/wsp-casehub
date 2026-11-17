# Situational Personality Modulation — Design Spec

Ref: casehubio/neocortex#712 — **Part 1: runtime consumption** (see §Scope)
Revision: 2 (adversarial review round 1 — every claim about existing code below was re-verified against the neocortex `main` tree in IntelliJ)

## Goal

Make a character's PID-5 personality *state* respond to the situation it is in (stress, flow, social context) and feed that state through the CDE so that appraisal, regulation, coping, search budget, capacity and expression parameters shift in an archetype-consistent way. Brief or mild exposure leaves no trace. Prolonged, repeated or extreme exposure leaves a *bounded*, slowly reversible trace in the character's resting personality (the elastic-band model).

## Scope

#712 asks for more than one deliverable. Its own scope line is "design the runtime consumption path first, then add extraction support." This spec is the runtime consumption path. #712 stays open as the parent until the deferred items below close.

| #712 requirement | Where |
|---|---|
| Stress mode, flow mode, social context (collaborative / competitive / hierarchical) | **This spec** |
| Per-character shadow rules carried in Eidos, with archetype-family defaults (Q1) | **This spec** (Part 0b, contract), implemented in casehubio/eidos#234 |
| CAPS-triggered activation (Q2) | **This spec** (§Detection: per-tick CAPS settlement over situation inputs, mediating units) |
| Prompt rendering of shadow context (Q3) | Answered: v1 is implicit (facet shifts through the CDE). Explicit `ShadowArchetypePromptSection` → #1059 |
| Biographical extraction of shadow patterns (Q4) | #1056 |
| Temporal drift (archetype salience across life phases) | #1057. Not the same thing as the elastic-band `home` deformation in this spec. |
| Narrative identity frame; classifying behavioural attractors as degraded archetype expression | #1058 (relates to #599) |
| CARMA: SEC evaluation aware of the situational mode | #1059 |

Prerequisites, designed here (Part 0) and tracked separately: #1055 (effective cognitive defaults + production Eidos baseline wiring) and casehubio/eidos#234 (`AgentDisposition.situationalProfiles`).

Pre-existing defects found during design, filed and not fixed here: #1060 (`PersonalityEvolutionOrchestrator` is never ticked), #1061 (`BehavioralSynthesisPhase` caches are agentId-keyed), #1062 (per-tick CAPS settlements are duplicated).

## Background

### What exists (verified)

**Personality → CDE.**
- `CognitiveDerivationEngine.derive(DescriptorView, eventRef, observer)` maps disposition axes, Jungian profile and PID-5 facets to `CognitiveDefaults` through 14+ pathways (#845).
- `deriveAndMerge(explicit)` merges with explicit-wins semantics: every non-null field of `explicit` is kept (`EmotionDefaults.mergeWith`, and `mergePersonalityProfile` keeps explicit facets whenever they are non-empty; `CognitiveDerivationEngine.java:297-322`).

**The Eidos → CDE path is not wired in production.**
- `EidosCognitiveProfileBridge.bridgeAll` has no production callers in neocortex or agency (only `EidosCognitiveProfileBridgeTest:188`).
- `CognitiveDefaultsRegistry.register` is called only by the bridge.
- There are no production `cognitive-profiles/*.yaml` files; the only two are test resources in `cognitive-index`.
- So every running agent gets `CognitiveDefaults.empty(agentId)`.
- Even if `bridgeAll` were called, re-registration is a no-op: the bridge passes the *previously merged* defaults as `explicit` (`EidosCognitiveProfileBridge.java:84-87`).
- The registry is `Map<agentId, CognitiveDefaults>` behind a volatile non-atomic copy-on-write (`CognitiveDefaultsRegistry.java:45,78-82`).
- Part 0a fixes all of this.

**Tick structure** (`CognitionCore.tick`, `:286-379`). Custom participants are ordered by `CognitionPhase`, appended per phase (`addParticipant`, `:1239`). Execution order:
1. mood tick #1
2. FOUNDATION
3. SOURCE
4. per-subject
5. drives
6. DERIVED (appraisal, regulation and search budget all read cognitive defaults here)
7. appraisal → mood bridge (`bridgeAppraisalToMood`), then mood tick #2 and drives tick #2
8. goals
9. TERMINAL

The tick's `EventRef` is `EventRef.tick(tickNumber)` (`:245`).

**Per-tick situation signals.**
- `AppraisalTickParticipant` classifies every observation into CAPS input activations through the `SituationClassifier`, every tick (`AppraisalTickParticipant.java:186-195`). It uses them only for channel weights. Implementations include `RuleBasedSituationClassifier` (vocabulary, no LLM) and `LlmSituationClassifier`.
- `AppraisalResult.secDimensions` carries relevance, novelty, urgency, conduciveness, controllability, adjustability, internal-standards and external-standards (`SecDimensions.java`). Conduciveness is bipolar [-1, 1]; the others are [0, 1] (`EmotionMapper.java:24-30`).
- PAD mood is [-1, 1] (`MoodState.validateAxis`). It decays toward a global baseline by elapsed time (`MoodOrchestrator.doTick`, `MoodDecay`, default τ = 4 h).

**CAPS.**
- `BehavioralSynthesisPhase` is a `ConsolidationPhase` (`@Priority(19)`). It settles only over unprocessed graduated memory nodes on the consolidation cadence. Its result cache is in-memory and agentId-keyed (#1061). It is not a per-tick signal.
- Per-tick settlements exist only inside `EmotionRegulationTickParticipant` (social-context inputs, conditional) and `CopingMicroLoop` (#1062).
- 19 of 51 CAPS output nodes have no incoming connections. These include all four learning outputs (`explore`, `consolidate`, `generalize`, `discriminate`), `submit`, `assert`, `perfectionism` and `risk_take` (#1050 / #1051).
- The `distrust` attractor is currently not produced (#1049).
- Mediating units (`BIS_activation`, `BAS_activation`, `FFFS_activation`, `threat_sensitivity`, `arousal_level`, `self_efficacy`, `other_safety`, …; `caps-topology.yaml:441-546`) are graded state variables.

**PersonalityEvolutionOrchestrator.** Not a tick participant. `tick` has no production callers. It acts only on Jungian function terms, and its `DispositionProfileStore` writes have no path back to the descriptor (#1060).

### Theoretical grounding

- **Whole Trait Theory** (Fleeson & Jayawickreme 2015).
  - A trait has two parts. The descriptive part is a density distribution of personality *states*. The explanatory part is the social-cognitive mechanisms that produce those states in situations.
  - This spec models both. `current` is the personality state. The CAPS settlement over situation features (Mischel & Shoda 1995, if-then signatures) is the explanatory mechanism that moves it.
  - This is integration, not double counting. CAPS output attractors select *behaviours* (regulation/coping strategy, approach/avoid). The facet state sets the continuous *cognitive parameters* the CDE derives. The detector reads CAPS mediating units, which are the "if" side, and never the output attractors.
- **Transactional stress** (Lazarus & Folkman 1984), operationalised through Scherer's stimulus evaluation checks. Stress means a situation appraised as relevant, goal-obstructive and beyond coping potential: relevance × negative conduciveness × low controllability.
- **Flow** (Csikszentmihalyi 1990; Nakamura & Csikszentmihalyi 2002). Flow is high challenge matched by high skill. It is an inverted-U in arousal, not a monotone signal.
- **State → trait consolidation** (TESSERA; Wrzus & Roberts 2017). Lasting change accrues from *repeated* state experiences through slower, reflective and associative processes. That is why the slow variables update in a consolidation phase, not per tick.
- **Set-point and partial return** (Jeronimus, Riese, Sanderman & Ormel 2014). Life-event effects on neuroticism are real, but partial and partly reversible. Lasting deformation is bounded and decays toward the baseline over months.
- **Mean-level change magnitude** (Roberts, Walton & Viechtbauer 2006). Lasting trait change is in the range of fractions of an SD over years. This bounds `D_max`.
- **Resilience and sensitisation.** Resilience is the common outcome after adversity and it recovers (Bonanno 2004). Repeated stress lowers the threshold and slows return (stress sensitisation; Post 1992). So resilience degrades with load and heals with rest, and it never ratchets to zero.
- **Archetype shadow forms.** Each of the 12 archetypes has a characteristic shadow expression (Pearson 1991, *Awakening the Heroes Within*; the 12 families are the Mark & Pearson 2001 brand archetypes refined into 48 sub-archetypes by Hartwell & Chen 2012). These sources fix the *direction* of each family's deltas (qualitative consensus). No source gives PID-5 *magnitudes*. Every magnitude is tagged `provenance: estimated` and tuned by eval, following the `caps-topology.yaml` `empirical / consensus / estimated` convention.

## Architecture

```
Eidos (class definition, per character)                 neocortex (runtime instance, per tenant × agent)
──────────────────────────────────────────               ──────────────────────────────────────────────────────────────
AgentDisposition.ampdFacets         (baseline) ──┐        CognitionCore.tick
AgentDisposition.situationalProfiles (rules)  ──┤          ├─ start: EffectiveCognitiveDefaults.updateBase(view)      [Part 0a]
   authored / biographical (#1056) or           │          ├─ FOUNDATION … DERIVED (read forAgent(tenant, agent))
   family-default (eidos#234)                   │          └─ TERMINAL: SituationalPersonalityParticipant
                                                └──────────►    detector → fast elastic update → updateFacets(...)  ─┐
                                                                state: MindMap node (cognitive subgraph)             │
                                                          ConsolidationScheduler                                      │
                                                            └─ SituationalDeformationPhase: home offset, resilience ◄┘
```

### Class / instance model

| Concept | Owner / store | Updated by | Cadence |
|---|---|---|---|
| Baseline facets | Eidos `AgentDisposition.ampdFacets` (0–100) | engram import | character creation |
| Situational profiles (deltas per mode) | Eidos `AgentDisposition.situationalProfiles` (0–100 deltas) | authored, extraction (#1056), or family default (eidos#234) | character creation / registration |
| Detector configuration + dynamics parameters | neocortex `cognition` resource `situational-personality.yaml` | deployment | static |
| `current` (personality state) | MindMap `situational-personality-state` node | tick participant | per tick |
| Mode intensities | same node | tick participant | per tick |
| Exposure / load accumulators | same node | tick (accrue), consolidation (drain) | both |
| `homeOffset` (lasting deformation) | same node | consolidation phase | per consolidation pass |
| `resilience` | same node | consolidation phase | per consolidation pass |
| Effective cognitive defaults | `EffectiveCognitiveDefaults` (tenant-scoped, in-memory, derived) | Part 0a | on base or facet change |

There is no separate "effective personality" concept. The CDE input is `current`, published as the facet overlay.

### Part 0a — Effective cognitive defaults (prerequisite, #1055)

The current registry cannot express "re-derive from new facets". It loses the explicit-vs-derived provenance, keys by agentId only, and is never fed in production. Patching a `refreshWithModulatedFacets` method onto it would inherit all three defects, so this part replaces it.

- **Explicit layer.** `CognitiveDefaultsRegistry` is reduced to the authored overrides: classpath and watched YAML (`CognitiveProfileWatcher`).
  - Immutable per reload, keyed by agentId. It is deployment configuration shared across tenants by construction, the same as `caps-topology.yaml`. It is not tenant data.
  - Profiles that pin `personalityProfile.facets` are rejected at load. Facets have exactly one source, Eidos (#845: "single path, no forks").
- **Effective layer.** New `EffectiveCognitiveDefaults` (`cognitive-index`, `@ApplicationScoped`).
  - Holds `ConcurrentHashMap<TenantAgentKey, Slot>`.
  - Each slot holds its inputs (`baseView: DescriptorView`, `facetOverlay: Optional<List<PersonalityFacet>>`) and the derived result.
  - One private re-derivation path: `merge(explicit(agentId), CDE.derive(baseView.withFacets(overlayOrBase), eventRef, observer))`. It runs inside `compute` on the slot, so there are no lost updates.
  - It never merges with a previously derived value.
  - Public API:
    - `CognitiveDefaults forAgent(String tenantId, String agentId)` — effective, else explicit-only, else `CognitiveDefaults.empty`.
    - `void updateBase(String tenantId, DescriptorView view, EventRef ref)` — no-op when `view` equals the stored base (record equality).
    - `void updateFacets(String tenantId, String agentId, List<PersonalityFacet> facets, EventRef ref)` — no-op when no base exists yet.
- **`CognitiveDerivationEngine.deriveAndMerge` is removed** and replaced by a pure `merge(explicit, derived)` with the same field-wise explicit-wins rule.
- **Decision — pinned fields are immune to situational modulation.** A field an author pins in explicit YAML (e.g. `regulationDefaults`) is an authoring override, and it wins over every derivation, situational or not. This is the existing semantics of the explicit layer, now applied to an explicit layer that really is explicit.
- **Baseline wiring.** At the start of `CognitionCore.tick`, before mood tick #1, when `context.descriptor() != null`, it calls `updateBase(tenantId, EidosCognitiveProfileBridge.toDescriptorView(descriptor), tickEventRef)`.
  - `bridgeAll` and `registerFromDescriptor` are deleted. The bridge keeps its single responsibility: `AgentDescriptor → DescriptorView` translation, including the 0–100 → 0–1 facet conversion.
  - New module edge: `cognition → cognitive-eidos`. There is no cycle; `cognitive-eidos` depends only on `cognitive-index` and `eidos-api`.
- **Consumer migration.** All `CognitiveDefaultsRegistry.forAgent / forAgentOrDefaults(agentId)` reads move to `EffectiveCognitiveDefaults.forAgent(tenantId, agentId)`.
  - About 108 production references across `cognition`, `cognitive-index`, `mindmap-intelligence` and `cognitive-eidos`.
  - Every consumer already has a tenantId in scope: tick context, `ConsolidationPhase.run(tenantId, …)`, or orchestrator calls.
  - Breaking by design; the migration is mechanical.
- **Observability.** Every re-derivation passes the tick `EventRef` and the session `DerivationObserver`. The existing `StageComputed` / `DataFlowed` journal entries then attribute *which facet shift changed which parameter* (production wiring of the journal bridge: #981).

### Part 0b — Situational profiles in Eidos (prerequisite, casehubio/eidos#234)

The design comment on #712 and #845 Step 0c say shadow *rules* are part of the character definition, carried in Eidos. Family-level tables in neocortex would make every Creator degrade identically and leave no place for biographical extraction (Q4). The contract:

- **eidos-api**:
  - `record AmpdFacetShift(String facet, int delta)` on the AMPD 0–100 scale (|delta| ≤ 50).
  - `record SituationalProfile(String mode, List<AmpdFacetShift> shifts, String provenance)`.
  - `AgentDisposition.situationalProfiles` (default empty).
- **Mode vocabulary** (closed): `stress`, `flow`, `social_collaborative`, `social_competitive`, `social_hierarchical`.
- **eidos-vocab**: `archetype-situational-profiles.yaml` holds 12 families × 5 modes = **60 family-default profiles**. Each entry carries `provenance` and `sources`. The examples below show the shape on the 0–100 delta scale.
- **eidos-runtime**: `SituationalProfileDeriver` runs after `ArchetypeDeriver`.
  - It resolves the free-form `archetype` string to a family: `ArchetypeTerm.valueOf(...).family()`, else `ArchetypeFamily.valueOf(...)`, else none.
  - For every mode the descriptor does not author, it adds the family default with `provenance: family-default:<FAMILY>`.
  - Override granularity is the whole mode, because an authored profile is a coherent unit.
- **Why family resolution lives in Eidos.** The archetype vocabulary is there. `eidos-vocab` has a compile-scope `quarkus-arc` dependency. neocortex `cognition` uses only `jakarta.enterprise.cdi-api` and must stay framework-neutral (a Spring variant exists), so it must not depend on `eidos-vocab`. neocortex never sees archetype strings. It reads `descriptor.disposition().situationalProfiles()`.
- **No profile, no modulation.** A null or unknown archetype with no authored profiles yields an empty list, and the participant does nothing for that agent. A universal "everyone under stress" profile is rejected: it would erase the archetype differentiation #845 built.
- **Secondary archetypes are not blended.** The descriptor carries one archetype. Individual nuance comes from per-character profiles (#1056), which subsume blending.

```yaml
# eidos-vocab: archetype-situational-profiles.yaml (shape; deltas on the AMPD 0–100 scale)
CREATOR:
  stress:
    description: "Obsessive self-validation, perfectionism paralysis"   # documentation only (narrative frame: #1058)
    provenance: estimated          # direction: consensus (Pearson 1991 Creator shadow); magnitude: estimated
    sources: [pearson-1991]
    shifts: {Anxiousness: +15, Emotional Lability: +20, Grandiosity: +10, Withdrawal: +10}
  flow:
    description: "Expansive creative expression, boundary dissolution"
    provenance: estimated
    sources: [csikszentmihalyi-1990, pearson-1991]
    shifts: {Risk Taking: +15, Impulsivity: +10, Grandiosity: +5, Anxiousness: -10}
  social_competitive:
    description: "Defensive posturing, dismissive of others' work"
    provenance: estimated
    sources: [pearson-1991]
    shifts: {Hostility: +10, Grandiosity: +15, Suspiciousness: +10}
  social_collaborative: {...}
  social_hierarchical:  {...}
CAREGIVER:
  stress:
    description: "Martyrdom, boundary collapse, resentful compliance"
    provenance: estimated
    sources: [pearson-1991]       # Caregiver shadow: martyr / enabler
    shifts: {Emotional Lability: +20, Anxiousness: +15, Callousness: -10, Withdrawal: -15}
  # ...
```

### Situational modes

```java
enum SituationalMode {
    STRESS(Group.AFFECTIVE), FLOW(Group.AFFECTIVE),
    SOCIAL_COLLABORATIVE(Group.SOCIAL), SOCIAL_COMPETITIVE(Group.SOCIAL), SOCIAL_HIERARCHICAL(Group.SOCIAL);
    enum Group { AFFECTIVE, SOCIAL }
}
```

- **Groups are mutually exclusive inside, orthogonal between.**
  - Stress and flow are opposite regions of the same challenge–skill space, so composing their deltas additively would be incoherent.
  - The three social frames describe one relational stance toward the current interlocutor.
  - In each group, only the engaged mode with the highest intensity contributes deltas. Losing modes keep decaying; they are not reset.
- At most two modes contribute at once, one affective and one social. Their deltas add, then the per-facet clamp applies (§Dynamics).
- The mode strings map 1:1 to the Eidos vocabulary.

### Detection

#### Signals

Each signal has a concrete source, range, normalisation, freshness rule and a **loop class**. A signal is *exogenous* if it does not depend on the modulated facets within the fast loop. It is *endogenous* if it does.

| Signal | Accessor | Raw range | Freshness | Loop class |
|---|---|---|---|---|
| Situation input activations | `AppraisalTickParticipant.currentInputActivations(agentId, tenantId)` — **new accessor** exposing the activations it already computes per tick (`:189-190`, after `RelationalBiasing`). Cleared at the start of its tick, absent when there is no observation or the classifier failed. | [0, 1] per input node | this tick only | exogenous — the classifier sees observation text plus relational biasing, never facets |
| CAPS mediating activations | per-tick `CapsEngine.settle(loadState(tenant, agent), situationInputs)` done by the participant; journaled as the existing `CognitiveEntry.CapsSettled` | unipolar [0, 1]; bipolar [-1, 1] (`caps-topology.yaml:441-546`) | this tick only | exogenous — weights are seeded once at `initializeAgent` and learned in consolidation; they are not a function of the modulated facets |
| Relational schema of the interlocutor | MindMap relational schema for `context.targetAgentId()` or `regulationContext.interactingAgents()`. The resolver is extracted into a shared `RelationalSchemaResolver`; it is duplicated today in `AppraisalTickParticipant` and `EmotionRegulationTickParticipant`. | trust [-1, 1], intimacy [0, 1], role | consolidation-maintained (`RelationalSchemaEvolutionPhase`) | exogenous |
| SEC dimensions | `AppraisalTickParticipant.currentResult(...).secDimensions()` | see §Background | this tick (DERIVED) | **endogenous** — appraisal weights and the LLM appraisal prompt read the facets |
| Post-appraisal mood | `MoodOrchestrator.currentMood(...)`, read in TERMINAL after mood tick #2 | [-1, 1] | this tick | **endogenous** — downstream of appraisal |

**Not used:**
- `BehavioralSynthesisPhase.lastSettlingResult`: consolidation cadence, arbitrarily stale, agentId-keyed (#1061).
- CAPS *output* attractors: these are behaviours rather than situation appraisal, and 19 of 51 are structurally dead (#1051).
- `CognitiveDefaults` appraisal weights: they are the modulation's own output.

**Missing sources.** A missing source is dropped and the remaining weights within its loop class are renormalised. A class with no available source scores 0. Without a `SituationClassifier`, therefore, no mode can engage and the feature is inert. `RuleBasedSituationClassifier` is the non-LLM option.

**Social gate.** Social modes require an interlocutor (`targetAgentId` or `interactingAgents`). Without one, the social exogenous score is 0.

#### Transforms

Every signal declares a transform that maps it into [0, 1]:

| Transform | Definition |
|---|---|
| `identity` | x (input already in [0, 1]) |
| `positive_part` | max(0, x) |
| `negative_part` | max(0, −x) |
| `rescale_bipolar` | (x + 1) / 2 |
| `complement` | 1 − x |
| `band(c, w)` | max(0, 1 − \|x − c\| / w) — inverted U |
| `challenge_skill(a, b)` | √(a·b) · (1 − \|a − b\|) — high only when both are high and matched |

A signal may be a `product` of transformed factors. Bipolar sources must declare a transform other than `identity`; this is checked at load.

#### Evidence, engagement and intensity

For each mode m, with `X_m` and `N_m` the weighted means of its exogenous and endogenous signals (each in [0, 1]):

```
E_m = w_X · X_m + w_N · N_m                     w_X + w_N = 1
I_m ← I_m · exp(−Δt / τ_fade,m)                 time-based fade (Δt = time since this agent's last tick)
if !engaged_m and E_m ≥ θ_on,m:  engaged_m ← true
if engaged_m:  I_m ← max(I_m, E_m)
if engaged_m and I_m < θ_off,m:  engaged_m ← false; I_m ← 0
```

- Hysteresis (θ_off < θ_on) prevents flicker at the boundary.
- The fade is time-based, so an idle agent leaves stress after real time has passed, not after a number of turns.

#### Lock-in invariant (stability against the feedback loop)

The loop exists. Modulated facets change appraisal weights (e.g. Anxiousness raises fear onset, `CognitiveDerivationEngine.java:712-722`) and the LLM appraisal prompt (`LlmAppraisalStrategy.java:225-236`). Those feed SEC values and mood, which are detector inputs. The design makes the loop *structurally* safe instead of relying on gain estimates:

- **Invariant: `w_N < θ_off`** for every mode. The loader rejects a configuration that violates it.
  - Consequence: with no exogenous evidence (`X_m = 0`), `E_m ≤ w_N < θ_off ≤ θ_on`. A mode cannot engage on self-generated evidence alone.
  - An engaged mode cannot be held: `I_m` decays by `τ_fade`, and the max with `E_m` cannot lift it back above `θ_off`.
  - Worst-case exit time once exogenous evidence stops: `τ_fade · ln(1 / θ_off)`, about 1.7 h for stress at defaults. This holds however strongly the modulation has amplified appraisal.
- While exogenous evidence *is* present, the endogenous term adds at most `w_N` to the evidence. That is a bounded mood-congruent appraisal bias (stressed people appraise more threat), which is the intended psychology.
- **Timing.** Modulation computed in TERMINAL at tick *t* is read by appraisal in DERIVED at tick *t+1*. The phase order enforces a one-tick delay.
- The Flow mirror loop raised in review (Risk Taking → `deriveMoodBaseline` → positive mood) does not reach the detector. `MoodOrchestrator` decays toward the global `MoodConfig.baseline()`, and the CDE mood baseline is consumed only by `MoodCongruentGoalAppraisal`. If that path is ever added, the invariant above still bounds it.

#### Configuration (`cognition/src/main/resources/situational-personality.yaml`)

```yaml
modes:
  stress:                                  # Lazarus & Folkman 1984; Scherer SEC
    group: affective
    exogenous:
      weight: 0.6
      signals:
        - {source: situation_input, nodes: [physical_threat, social_threat, psychological_threat, unpredictable_danger], aggregate: max, transform: identity, weight: 0.35}
        - {source: situation_input, nodes: [failure, scarcity, powerlessness, rejection, betrayal, abandonment], aggregate: max, transform: identity, weight: 0.25}
        - {source: caps_mediating, nodes: [FFFS_activation, BIS_activation, threat_sensitivity], aggregate: mean, transform: identity, weight: 0.40}
    endogenous:
      weight: 0.4
      signals:
        - {source: sec, product: [{dim: relevance}, {dim: conduciveness, transform: negative_part}, {dim: controllability, transform: complement}], weight: 0.60}
        - {source: mood, axis: pleasure, transform: negative_part, weight: 0.25}
        - {source: mood, axis: arousal,  transform: positive_part, weight: 0.15}
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT2H
  flow:                                    # Csikszentmihalyi 1990; Nakamura & Csikszentmihalyi 2002
    group: affective
    exogenous:
      weight: 0.6
      signals:
        - {source: situation_input, nodes: [mastery, competence_recognition, choice_available, agency_granted], aggregate: max, transform: identity, weight: 0.5}
        - {source: caps_mediating, nodes: [BAS_activation], transform: identity, weight: 0.3}
        - {source: caps_mediating, nodes: [self_efficacy], transform: positive_part, weight: 0.2}
    endogenous:
      weight: 0.4
      signals:
        - {source: sec, product: [{dim: relevance}, {dim: conduciveness, transform: positive_part}, {challenge_skill: [urgency, controllability]}], weight: 0.60}
        - {source: mood, axis: arousal,  transform: "band(0.25, 0.6)", weight: 0.25}
        - {source: mood, axis: pleasure, transform: positive_part, weight: 0.15}
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT45M
  social_collaborative:
    group: social
    exogenous:
      weight: 0.6
      signals:
        - {source: situation_input, nodes: [acceptance, sharing, perceived_responsiveness], aggregate: max, transform: identity, weight: 0.4}
        - {source: relational, field: trust, transform: rescale_bipolar, weight: 0.35}
        - {source: caps_mediating, nodes: [other_safety, other_reliability], aggregate: mean, transform: positive_part, weight: 0.25}
    endogenous:
      weight: 0.4
      signals:
        - {source: mood, axis: pleasure, transform: positive_part, weight: 1.0}
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT30M
  social_competitive:
    group: social
    exogenous:
      weight: 0.6
      signals:
        - {source: situation_input, nodes: [competition, exclusion, dominance], aggregate: max, transform: identity, weight: 0.45}
        - {source: relational, field: trust, transform: negative_part, weight: 0.30}
        - {source: caps_mediating, nodes: [other_safety, other_reliability], aggregate: mean, transform: negative_part, weight: 0.25}
    endogenous:
      weight: 0.4
      signals:
        - {source: sec, product: [{dim: relevance}, {dim: external-standards, transform: complement}], weight: 0.6}
        - {source: mood, axis: arousal, transform: positive_part, weight: 0.4}
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT30M
  social_hierarchical:
    group: social
    exogenous:
      weight: 0.6
      signals:
        - {source: situation_input, nodes: [power_asymmetry, controlled, submission], aggregate: max, transform: identity, weight: 0.6}
        - {source: relational, field: role_asymmetry, transform: identity, weight: 0.4}   # 1 when the schema role is a recognised superior/subordinate role
    endogenous:
      weight: 0.4
      signals:
        - {source: mood, axis: dominance, transform: negative_part, weight: 1.0}
    threshold_on: 0.50
    threshold_off: 0.42
    fade_tau: PT30M
```

**Load-time validation** fails startup on any of the following:
- Mode keys not in `SituationalMode`.
- `w_X + w_N ≠ 1`, or signal weights within a class that do not sum to 1.
- **`w_N ≥ θ_off`.**
- `θ_off ≥ θ_on`.
- A `situation_input` node not in the loaded `CapsTopology` input set.
- A `caps_mediating` node not in the mediating set, or with no incoming connection (guards against #1051-style orphans).
- A bipolar source with transform `identity`.
- Unknown SEC dimension or mood axis.

### Dynamics

All facet quantities are on the 0–1 scale. Eidos values (0–100) are divided by 100 once, at the bridge. Per-agent state covers only the facets **present in the agent's baseline** (`F_agent`; see §Facet coverage).

| Symbol | Meaning |
|---|---|
| `b_f` | baseline (from the descriptor, every tick; not stored) |
| `o_f` | `homeOffset`, the lasting deformation; `home_f = clamp01(b_f + o_f)` |
| `c_f` | `current`, the personality state |
| `r_f` | `resilience` ∈ [r_min, 1] |
| `A_f`, `L_f` | signed exposure ∫(c−home)dt and load ∫\|c−home\|dt since the last consolidation (facet·hours) |

Storing the deformation as an **offset** from the baseline, not as an absolute `home`, means a corrected or re-extracted Eidos baseline keeps the character's lived deformation instead of silently discarding it.

#### Fast update — `SituationalPersonalityParticipant`, TERMINAL, every tick

Here `Δt` is the time since this agent's state was last updated, read from an injected `Clock`. `d_f = c_f − home_f`, and `k_f = r_f / τ_return`.

1. **Relax (time-based) and accrue exposure, exactly.**
   - `A_f += d_f·(1−e^{−k_fΔt})/k_f`
   - `L_f += |d_f|·(1−e^{−k_fΔt})/k_f`
   - `d_f ← d_f·e^{−k_fΔt}`

   The closed-form integral makes exposure independent of tick rate. An agent idle for a week returns home (`e^{−168/6}` ≈ 0), and its mood has decayed too.
2. **Detect** (§Detection). This updates the intensities and the group winners.
3. **Target.**
   - `Δ_f = clamp(Σ_{m ∈ winners} I_m · δ_{m,f}, −S_max, +S_max)`
   - `t_f = clamp01(home_f + Δ_f)`
4. **Pull (event-based, only when a winner exists).** `c_f ← c_f + α_pull·(t_f − c_f)`.
   - Each turn is an exposure event, so pull is per turn. Its effect saturates at the target within about 9 turns at α = 0.3. Lasting effects are driven by `A`/`L`, which are real-time integrals, never by turn count.
5. **Publish (quantised).**
   - If `max_f |c_f − published_f| ≥ ε_publish`, or a winner was gained or lost, call `EffectiveCognitiveDefaults.updateFacets(tenant, agent, facets, tickEventRef)`.
   - Facets with `|c_f − b_f| ≥ ε_publish` carry `origin = "situational"`; the rest keep `"eidos"`.
   - Then set `published = c` and journal `SituationalModulation`.
   - Refreshed defaults take effect **next tick**, because DERIVED has already read this tick's defaults.
6. **Persist** the state node when anything in it changed. That is one MindMap node write per active agent per turn, the same order as `EmotionRegulationMindMapAdapter.write`.

**Anchoring the target to `home`, not to the immutable baseline.** A character whose resting Anxiousness has been permanently raised should, under stress, become anxious *relative to who they now are*. Anchoring to `b_f` would make stress partially undo the deformation (target = b + Δ < home + Δ), which is incoherent. The reviewer's concern was unbounded drift. That is closed by the bounds below, not by moving the anchor.

#### Slow update — `SituationalDeformationPhase` (`ConsolidationPhase`, `cognition` module)

This mirrors `DriveAdaptationPhase`: it runs per tenant over the cognitive subgraphs and is idempotent per pass. Here `T` is the time since `lastConsolidatedAt`. For each facet, in order:

```
fatigue:   r_f ← max(r_min, r_f − L_f / τ_fatigue)                         sensitisation (Post 1992)
heal:      r_f ← 1 − (1 − r_f) · exp(−T / τ_heal)                          resilience recovers (Bonanno 2004)
drift:     step = (1 − r_f) · A_f / τ_drift
           if sign(step) == sign(o_f):  step ← step · (1 − |o_f| / D_max)     soft saturation toward the bound
           o_f ← clamp(o_f + step, −D_max, +D_max)                            (clamp is a numerical guard)
set-point: o_f ← o_f · exp(−T / τ_setpoint)                                 partial return (Jeronimus et al. 2014)
reset:     A_f ← 0, L_f ← 0, lastConsolidatedAt ← now
```

- It journals one `SituationalDeformationConsolidated` entry per agent per pass. That gives the audit point for lasting change.
- The resilience term replaces the round-1 `elasticity` / `elasticity_floor` pair. The floor version had three problems:
  - Elasticity always recovered to 1.0, so the floor never limited recovery.
  - Its initial value was undefined.
  - It ratcheted to zero.

  Now repeated load lowers resilience, which slows return (`k_f = r_f/τ_return`). That raises exposure, which drives drift. Rest heals resilience, and `r_min > 0` forbids permanent loss of the ability to recover.
- **Time-scale separation.** This is why D1 rejected extending PEO. The tick participant owns only fast state: `c`, intensities and accumulation. Lasting change happens only in consolidation, from accumulated exposure.

#### Concurrency and ownership

- `SituationalStateService` (`cognition`) owns:
  - the in-memory slot cache `ConcurrentHashMap<TenantAgentKey, Slot>`;
  - a per-key `ReentrantLock` (the `MoodOrchestrator` pattern);
  - MindMap load and persist.
- Both the participant and the consolidation phase go through it, so a consolidation pass can never interleave with a tick on the same agent.
- Slots idle longer than 24 h are persisted and evicted, then reloaded from MindMap on the next access.
- Crash semantics: state is persisted at the end of every tick that changes it, so a crash loses at most the in-flight tick.

#### Invariants (all checked by property tests)

| Invariant | Why it holds |
|---|---|
| `I_m ∈ [0, 1]` | `E_m` is a convex combination of [0,1] terms; fade is multiplicative |
| `\|Δ_f\| ≤ S_max` | explicit clamp |
| `c_f ∈ [0, 1]` | `t_f` and `home_f` are clamped to [0,1]; relaxation and pull are convex combinations |
| `\|o_f\| ≤ D_max` | soft saturation plus guard clamp; set-point decay only shrinks it |
| `\|c_f − b_f\| ≤ D_max + S_max` | follows from the two above |
| `r_f ∈ [r_min, 1]` | fatigue floor; healing is a convex move toward 1 |
| no mode engaged on endogenous evidence alone | `w_N < θ_off` (validated) |

`PersonalityFacet` rejects scores outside [0, 1] and `safeRun` swallows exceptions. The clamps therefore live in the state machine, so an out-of-range value can never be constructed.

#### Worked example (defaults; Creator, stress, Emotional Lability δ = +0.20)

- **One intense 100-turn stressful conversation (~1.7 h, I ≈ 1).**
  - `c` reaches `home + 0.20` within about 9 turns.
  - After the conversation it relaxes with τ = 6 h.
  - `L ≈ 0.2·1.7 + 0.2·6 ≈ 1.5`, `A ≈ 1.5` facet·h.
  - At the next consolidation: `r` 1.00 → 0.96, `o` += 0.04·1.5/50 ≈ **+0.001**.

  Under the round-1 equations the same conversation permanently shifted the facet by ≈ 0.20 and saturated it after about 225 turns.
- **The same conversation daily for 30 days.**
  - Daily fatigue (≈ 0.038/day) outpaces healing ((1 − r)·0.033/day), so `r` falls to ≈ 0.42 by day 21 and ≈ 0.27 by day 30.
  - As (1 − r) grows, drift accelerates and saturates toward `D_max`. Integrated over 30 days this gives **o ≈ +0.09–0.10**: a lasting shift of roughly half a PID-5 facet SD.
- **Then 6 months without stress.** `r` heals toward 1 (τ = 30 d), and `o` decays by `e^{−1}` (τ_setpoint = 180 d). The return is partial, as set-point findings predict.

### Facet coverage and initialisation

- **Scale.** All runtime state is on [0, 1]. The 0–100 → 0–1 conversion happens once, at the bridge, for both baseline facets and profile deltas.
- **Only baseline facets are modulated.** `F_agent` is the set of facets present in `ampdFacets`.
  - Profile shifts for facets missing from the baseline are ignored and listed in the journal entry's `skippedFacets`.
  - No 0.5 seeding. The CDE Category A methods fall back to `DispositionAxes` when the facet list is empty (#845 Part 2), and treat absent facets as 0.5 when it is not. Adding facets would therefore switch derivation regimes, a discontinuity unrelated to the situation.
  - Agents with no `ampdFacets` are not modulated. The participant journals the "inactive: no baseline facets" reason once per slot load.
- **Initial state.** On the first tick for (tenant, agent): `c_f = b_f`, `o_f = 0`, `r_f = 1`, `A_f = L_f = 0`, all intensities 0, `updatedAt = lastConsolidatedAt = now`.
- **Provenance.** Shifted facets carry `origin = "situational"`. `LlmAppraisalStrategy` already renders facet origin into the appraisal prompt, so the appraising LLM can see which traits are situationally elevated. This is a deliberate, mild implicit cue. Explicit CARMA context is #1059.
- **Discrete thresholds.** `PersonalityFacet.isHigh / isLow` (0.7 / 0.3) gate discrete CDE branches. Quantised publishing (ε = 0.01) plus mode hysteresis limits flicker. Eval measures it (§Eval).

### Persistence

The state lives in a **MindMap node**, `cognitiveKind = situational-personality-state`, in the agent's cognitive subgraph (subgraph resolution as in `EmotionRegulationMindMapAdapter` and `DriveAdaptationPhase`). Properties:
- `agent-id`, `schemaVersion`, `updatedAt`, `lastConsolidatedAt`
- per facet: `facet.<Name>.current`, `.homeOffset`, `.resilience`, `.exposure`, `.load`
- per mode: `mode.<mode>.intensity`, `.engaged`

This reverses round-1 D5 (a separate SQLite `SituationalPersonalityStore`). The reasons:
1. **It is the documented architecture.** #845 Step 0c: "MindMap stores runtime personality state — situational shifts, stress degradation (#712) … The CURRENT shift state lives in MindMap". The #712 design comment and the published two-lenses-on-personality note say the same. Round 1 misquoted #845 to justify the opposite.
2. **The slow update is a consolidation phase, and consolidation phases operate on MindMap** (`ConsolidationPhase.run(tenantId, …)`, `DriveAdaptationPhase`'s `drive-intensity` nodes). A separate store would give consolidation a second persistence dependency for no gain.
3. **Tenancy and backends come for free.** Every `MindMapStore` call is tenant-scoped, and backend selection (in-memory / SQLite) already follows the MindMap SPI's CDI priority ladder. A new store would need its own SPI, an in-memory `@DefaultBean` and a SQLite backend, all duplicating MindMap.

### Pipeline integration

- `SituationalPersonalityParticipant` registers via `CognitionCore.addParticipant(CognitionPhase.TERMINAL, …)`. TERMINAL is the only phase that sees post-appraisal mood and this tick's appraisal.
- **Needs from the tick.**
  - `context.descriptor()` must be non-null; otherwise the participant does nothing.
  - `CognitionTickContext` gains `EventRef tickEventRef` (cognition-api) so participants can journal and re-derive under the tick's reference.
- **Collaborators.** `SituationalStateService`, `AppraisalTickParticipant` (input activations + result), `MoodOrchestrator`, `CapsEngine` (optional), `RelationalSchemaResolver`, `EffectiveCognitiveDefaults`, `CognitiveJournal`, `Clock`.
- **Per-tick CAPS settlement.**
  - When situation inputs exist and a `CapsEngine` is present, the participant runs `loadState` + `settle` itself.
  - If `loadState` returns null (the agent has no CAPS state yet), the `caps_mediating` signals are dropped (renormalised); it does not initialise CAPS.
  - This is the third independent per-tick settlement (regulation and coping are the others). Consolidating them is #1062.

### Observability

- **Journal entries** (new `CognitiveEntry` records, cognition-api):
  - `SituationalModulation(String tenantId, String agentId, EventRef eventRef, Instant timestamp, Map<SituationalMode, Double> intensities, Set<SituationalMode> winners, Map<String, Double> evidence, Map<String, Double> compositeDelta, Map<String, Double> publishedFacets, List<String> skippedFacets)`. Recorded on publish or on a winner change only, not every tick.
  - `SituationalDeformationConsolidated(String tenantId, String agentId, EventRef eventRef, Instant timestamp, Map<String, Double> homeOffsetBefore, Map<String, Double> homeOffsetAfter, Map<String, Double> resilienceBefore, Map<String, Double> resilienceAfter)`. One per agent per consolidation pass.
  - The per-tick settlement is journaled with the existing `CapsSettled`.
- **Derivation attribution.** Every `updateFacets` re-derivation carries the tick `EventRef` and the `DerivationObserver`, so `StageComputed` / `DataFlowed` show e.g. "Anxiousness 0.62 → 0.77 moved `fearOnsetThreshold` 1.31 → 1.36".
- **Pipeline trace.** `PipelineTrace` gains `SituationalPersonalitySnapshot(intensities, winners, evidence, profileProvenance, baselineFacets, currentFacets, homeOffsets, resilience)`. `CognitionCore.currentTrace(...)` (`:1095-1152`) gets it from `SituationalStateService`.

### Relationship to PersonalityEvolutionOrchestrator

- Round 1 claimed the bridge merges PEO's Jungian weights with modulated facets. That claim is withdrawn.
- PEO is never ticked, its profile-store writes do not reach the descriptor, and it never reads facets (#1060).
- The correct convergence point now exists: `EffectiveCognitiveDefaults` derives from `baseView` and `facetOverlay`. `baseView` carries the Jungian `dispositionProfile` from the descriptor; the overlay carries situational facets.
- Once #1060 propagates evolved profiles into the descriptor, `updateBase` re-derives with the current facet overlay intact. No merge logic is needed anywhere else.

## Parameters

Detector parameters are per mode in `situational-personality.yaml` (above). Dynamics parameters live in the same file under `dynamics:`. All are `provenance: estimated` and tuned by eval, except where the source column says otherwise.

| Parameter | Default | Unit | Meaning | Source / rationale |
|---|---|---|---|---|
| `alpha_pull` | 0.30 | per turn | fraction of the gap to the target closed per turn while a mode wins | estimated; saturates within ~9 turns |
| `tau_return` | 6 h | time | return time constant at full resilience (scaled by 1/r) | estimated; slower than mood (4 h): personality states outlast affect |
| `S_max` | 0.25 | facet | max composite situational shift per facet | estimated; ≈ 1 facet SD |
| `D_max` | 0.10 | facet | max lasting deformation per facet | Roberts et al. 2006; Jeronimus et al. 2014 (≈ ½ SD) |
| `tau_fatigue` | 40 | facet·h | load needed to lose all resilience | estimated (worked example) |
| `tau_heal` | 30 d | time | resilience recovery time constant | estimated; Bonanno 2004 (recovery is the norm) |
| `r_min` | 0.2 | — | resilience floor | forbids the round-1 ratchet to zero |
| `tau_drift` | 50 | h | exposure → deformation scale | estimated (worked example) |
| `tau_setpoint` | 180 d | time | decay of lasting deformation toward baseline | Jeronimus et al. 2014 |
| `epsilon_publish` | 0.01 | facet | quantum for re-derivation and journaling | below PID-5 measurement resolution |
| per mode `threshold_on / threshold_off` | 0.50 / 0.42 | — | engagement hysteresis | must satisfy `w_N < θ_off < θ_on` |
| per mode `fade_tau` | stress 2 h, flow 45 min, social 30 min | time | intensity fade | estimated; social frames end with the interlocutor |
| per mode `w_X / w_N` | 0.6 / 0.4 | — | exogenous / endogenous evidence split | lock-in invariant |

## Testing

### Unit (pure functions)

- **Detector.**
  - Each transform, including `band` and `challenge_skill`.
  - Renormalisation when a source is missing.
  - Social gate.
  - Group exclusion (stress + flow evidence → only the stronger contributes).
  - Hysteresis at the boundary.
  - Time-based fade.
- **Lock-in.**
  - With `X = 0` and `N = 1` sustained, the mode never engages.
  - Engaged at `I = 1`, then `X → 0` and `N = 1`: the mode disengages within `τ_fade·ln(1/θ_off)`.
- **Bounded intensity under a constant mild stressor** (review R1-07c). Constant `X = 0.55`, endogenous fed back through a stub that maps facet shift to SEC: `I` converges and stays below `w_X·0.55 + w_N`.
- **Dynamics.**
  - Property-based: random sequences of `Δt`, intensities and consolidation passes never violate the invariants table.
  - Exact-integral exposure: the same real-time exposure at 1 turn/s and at 1 turn/min yields `A` within 1 %.
  - Idle week: `|c − home| < 0.01`.
  - The worked-example numbers within tolerance.
  - Facets with δ = 0 accrue no load.
- **Slow update.**
  - Fatigue, heal, saturation and set-point each in isolation.
  - `r` never below `r_min`, `|o|` never above `D_max`.
- **Config validation.** Every rejection rule in §Detection, including the `w_N ≥ θ_off` and orphan-mediating-node rules.
- **Part 0a.**
  - Re-derivation with changed facets changes appraisal weights and regulation defaults (regression for the round-1 no-op).
  - A pinned explicit field survives.
  - An explicit profile with facets is rejected.
  - Two tenants with the same agentId are independent.
  - Concurrent `updateFacets` loses no update.
  - `updateBase` with an equal view is a no-op.
- **Part 0b** (eidos#234).
  - Term → family, family name, unknown → empty.
  - An authored mode overrides the family default.
  - Validation rules.

### Integration

- **End to end.** `RuleBasedSituationClassifier` + `SqliteCapsEngine` + in-memory MindMap + a real `CognitionCore`, all wired through Part 0a.
  1. A threat observation engages stress.
  2. At tick t+1, `AppraisalTickParticipant` reads changed appraisal weights and `EmotionRegulationTickParticipant` reads changed regulation defaults from `EffectiveCognitiveDefaults`.
  3. Neutral observations: the mode disengages and `c` relaxes.
  4. `SituationalModulation`, `CapsSettled` and `StageComputed` entries are present with the tick `EventRef`.
- **Persistence.** Evict the slot, reload from MindMap: identical state. Restart over SQLite MindMap: state survives.
- **Consolidation.** A consolidation pass during an active tick stream: no interleaving (lock), accumulators drained exactly once.
- **Tenancy.** The same agentId in two tenants: stress in tenant A leaves tenant B's state and effective defaults untouched.

(The round-1 "PersonalityEvolutionOrchestrator reads evolved facets" test is removed. That behaviour does not exist; see #1060.)

### Eval (pre-registered falsification criteria)

- **Differentiation.** Run the 12 families through an identical stress scenario. The mean pairwise distance between their normalised effective `CognitiveDefaults` vectors must be *greater* during stress than at baseline (paired test, p < 0.05). If stress homogenises characters, the design is falsified.
- **Coherence.** Judges rate archetype-consistency of responses under stress. The modulated run must score ≥ the unmodulated run (same seeds, same scenario) by ≥ 0.5 on a 5-point scale. Otherwise the facet-shift mechanism adds nothing observable.
- **Recovery.**
  - A single 30-turn stress episode plus 24 h simulated idle: `max_f |c_f − b_f| ≤ 0.02`.
  - 30 daily episodes: stressed facets end with `o ∈ [0.03, D_max]`.
  - Facets the profile does not touch end with `|o| < 0.005`.
- **Flicker.** At most one `isHigh`/`isLow` flip per facet per mode episode.

## Implementation ordering

| # | Step | Depends on |
|---|---|---|
| 0a | `EffectiveCognitiveDefaults`, explicit-layer split, `merge`, consumer migration, baseline wiring in `CognitionCore` (#1055) | — |
| 0b | Eidos `situationalProfiles` contract, `SituationalProfileDeriver`, 60 family-default profiles with provenance (casehubio/eidos#234). Profile authoring needs literature research and runs in parallel with 1–4. | — |
| 1 | `situational-personality.yaml` + loader/validator (needs the `CapsTopology` for node validation) | — |
| 2 | Dynamics pure functions (fast + slow) | — |
| 3 | Detector pure function | 1 |
| 4 | `AppraisalTickParticipant.currentInputActivations`, `RelationalSchemaResolver` extraction, `CognitionTickContext.tickEventRef` | — |
| 5 | `SituationalStateService` (cache, locks, MindMap persistence) | 2 |
| 6 | `SituationalPersonalityParticipant` (TERMINAL) | 0a, 0b, 1–5 |
| 7 | `SituationalDeformationPhase` | 2, 5 |
| 8 | Journal entries, `PipelineTrace` snapshot, `currentTrace` aggregation | 6, 7 |
| 9 | Eval | all |

Steps 0a, 0b, 1, 2 and 4 are independent.

## References

- casehubio/neocortex#712 — parent issue. Deferred: #1056, #1057, #1058, #1059. Prerequisites: #1055, casehubio/eidos#234. Related defects: #1060, #1061, #1062, #1049, #1050 / #1051, #981, #599.
- casehubio/neocortex#845 — CDE PID-5 facet upgrade. `examples/docs/specs/issue-845-cde-pid5-facets/2026-11-13-cde-pid5-facet-upgrade-design.md` (Step 0c: MindMap holds runtime personality state).
- `neocortex/docs/research/era7-personality-trait-integration-research.md`
- Code: `CognitiveDerivationEngine`, `CognitiveDefaultsRegistry`, `EidosCognitiveProfileBridge`, `CognitionCore`, `AppraisalTickParticipant`, `EmotionRegulationTickParticipant`, `MoodOrchestrator`, `DriveAdaptationPhase`, `BehavioralSynthesisPhase`, `caps-topology.yaml`, eidos `ArchetypeDeriver`, `ArchetypeTerm`, `ArchetypeFamily`.
- Bonanno, G. A. (2004). Loss, trauma, and human resilience. *American Psychologist*, 59(1), 20–28.
- Csikszentmihalyi, M. (1990). *Flow: The Psychology of Optimal Experience*. Harper & Row.
- Fleeson, W. & Jayawickreme, E. (2015). Whole Trait Theory. *Journal of Research in Personality*, 56, 82–92.
- Hartwell, M. & Chen, J. (2012). *Archetypes in Branding*. — 48 sub-archetypes / 12 families (taxonomy only).
- Jeronimus, B. F., Riese, H., Sanderman, R. & Ormel, J. (2014). Mutual reinforcement between neuroticism and life experiences. *Journal of Personality and Social Psychology*, 107(4), 751–764.
- Jung, C. G. (1959). *The Archetypes and the Collective Unconscious*. CW 9i.
- Krueger, R. F. et al. (2012). Initial construction of a maladaptive personality trait model and inventory for DSM-5 (PID-5). *Psychological Medicine*, 42(9).
- Gore, W. L. & Widiger, T. A. (2013). The DSM-5 dimensional trait model and five-factor models of general personality. *Journal of Abnormal Psychology*, 122(3).
- Lazarus, R. S. & Folkman, S. (1984). *Stress, Appraisal, and Coping*. Springer.
- Mark, M. & Pearson, C. S. (2001). *The Hero and the Outlaw*. McGraw-Hill.
- Mischel, W. & Shoda, Y. (1995). A cognitive-affective system theory of personality. *Psychological Review*, 102(2), 246–268. — CAPS: if-then situational signatures (the explanatory mechanism here, not the state-shift model).
- Nakamura, J. & Csikszentmihalyi, M. (2002). The concept of flow. In *Handbook of Positive Psychology*.
- Pearson, C. S. (1991). *Awakening the Heroes Within*. HarperCollins. — shadow forms of the 12 archetypes (direction of family deltas).
- Post, R. M. (1992). Transduction of psychosocial stress into the neurobiology of recurrent affective disorder. *American Journal of Psychiatry*, 149(8), 999–1010.
- Roberts, B. W., Walton, K. E. & Viechtbauer, W. (2006). Patterns of mean-level change in personality traits across the life course. *Psychological Bulletin*, 132(1), 1–25.
- Scherer, K. R. (2001). Appraisal considered as a process of multilevel sequential checking. In *Appraisal Processes in Emotion*.
- Wrzus, C. & Roberts, B. W. (2017). Processes of personality development in adulthood: the TESSERA framework. *Personality and Social Psychology Review*, 21(3), 253–277.
