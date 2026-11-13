# Decisions — #845 CDE PID-5 Facet Upgrade

## D1: Extend, not rewrite

**Choice:** Extend existing derivation methods to accept PID-5 facets alongside current inputs — not replace Jungian or disposition inputs wholesale.
**Alternatives:**
- Full rewrite to PID-5 only — loses cognitive style dimension that Jungian functions capture (HOW vs WHAT)
- No change — leaves 6 methods on coarse categorical inputs when continuous scores are available
**Rationale:** PID-5 facets measure pathological personality traits (WHAT is exhibited). Jungian cognitive functions measure processing style (HOW information is handled). These are complementary dimensions, not competing representations. The PID-5 AMPD framework was designed to complement, not replace, existing personality models (Krueger et al. 2012). Category A methods (DispositionAxes) measure trait dimensions that PID-5 directly covers — upgrade these. Category B methods (WeightedTerms) measure cognitive style — keep Jungian base, add facet modulation.
**Trade-offs:** Maintaining two input channels adds complexity. Mitigated by the reference pattern (`deriveSearchBudgetDefaults`) already accepting both.
**Sources:** Krueger et al. (2012) *Psychological Medicine* 42(9):1879-1890; Gore & Widiger (2013) *Journal of Abnormal Psychology* 122:816-821
**Exploration:** deep-analysis
**Status:** captured

## D2: Fix the bridge first

**Choice:** Build the missing `EidosCognitiveProfileBridge` → `PersonalityFacet` pipeline before upgrading CDE methods.
**Alternatives:**
- Upgrade CDE methods first, bridge later — existing facet-accepting methods are dead code in production regardless
- Skip bridge, use only YAML profiles — limits facets to test/eval contexts only
**Rationale:** The biography pipeline (`extract_personality.py`) independently scores 13 AMPD facets and writes them to MindMap properties (`personality-ampd-*`). `EidosCognitiveProfileBridge.toDescriptorView()` currently passes `List.of()` for `personalityFacets`. Fixing this unlocks the 5 existing facet-accepting methods AND the new ones. Scale conversion: 0-100 (MindMap) → 0.0-1.0 (PersonalityFacet).
**Trade-offs:** Couples the CDE upgrade to the bridge fix. Acceptable because both are needed for correctness.
**Sources:** `EidosCognitiveProfileBridge.java:110`, `PersonalityProfileHandler.java`, `extract_personality.py`
**Exploration:** quick
**Status:** captured

## D3: Category A — PID-5 replaces DispositionAxes lookups

**Choice:** For the 6 DispositionAxes methods, replace categorical string lookups with continuous `facetScore()` modulation. Keep DispositionAxes as fallback when facets are empty.
**Alternatives:**
- Keep DispositionAxes as primary — discards resolution (0.6 vs 0.9 collapsed to same bucket)
**Rationale:** DispositionAxes provide 3-4 categorical levels per dimension. PID-5 facets provide continuous [0,1] scores measuring the same trait dimensions. The `deriveSearchBudgetDefaults` reference pattern already demonstrates this: facets as primary, axes as fallback for uncovered dimensions.
**Trade-offs:** Behavioral outputs will shift for characters with facet scores near category boundaries. Eval framework catches regressions.
**Sources:** `CognitiveDerivationEngine.java` lines 340-592 (6 methods); reference pattern at lines 765-844
**Exploration:** quick
**Status:** captured

## D4: Category B — Facet modulation of Jungian-derived values

**Choice:** For the 5 WeightedTerms methods, keep Jungian function weights as the base computation. Add PID-5 facet modulation where an empirically validated interaction exists between a PID-5 facet and the cognitive dimension being derived.
**Alternatives:**
- Full facet rewrite — PID-5 doesn't measure cognitive style; compositions would be theoretically weaker than Jungian for these dimensions
- No modulation — misses validated interactions (e.g., Impulsivity degrades working memory)
**Rationale:** PID-5 and Jungian measure complementary dimensions. The Big Five mediates between them: MBTI→FFM (McCrae & Costa 1989) and FFM→PID-5 (Gore & Widiger 2013) are both peer-reviewed. This creates a validated dimension-level bridge. Function-level extensions are first-principles reasoning grounded in that bridge. The goal is characters that feel real — eval coherence scores are the validation, not clinical diagnosis.

Two-tier modulation approach:

**Tier 1 — Peer-reviewed direct evidence:**
| Facet | Modulates | Direction | Citation |
|---|---|---|---|
| Impulsivity | working memory span (deriveCapacity) | degrades | Heydarloo et al. (2024) *Memory*; Kofler et al. (2024) *Frontiers in Psychiatry* |
| Callousness | relationship bias (deriveExtractionBias) | reduces | Blair (2005, 2008, 2013) *Nature Reviews Neuroscience* |
| Anxiousness | fear weight (deriveAppraisalWeights) | amplifies | Bar-Haim et al. (2007) *Psychological Bulletin* — 172-study meta-analysis, d=0.45 |

**Tier 2 — First-principles extensions via FFM bridge:**
| Facet | Modulates | Direction | Reasoning chain |
|---|---|---|---|
| Emotional Lability | affect sensitivity (deriveExtractionBias) | amplifies | High lability = high Neuroticism (FFM) = amplified emotional reactivity |
| Risk Taking | fear weight (deriveAppraisalWeights) | reduces | High risk = Disinhibition domain → reduced threat sensitivity |
| Impulsivity | urgency (deriveAppraisalWeights) | amplifies | Disinhibition domain → immediate threat/opportunity weighting |
| Anxiousness | associative reach (deriveCapacity) | narrows | Negative Affectivity → attentional narrowing (Bar-Haim adjacent) |
| Withdrawal | connective bias (deriveGraphStructure) | reduces | Detachment domain → reduced social node connectivity |

Tier 2 modulations are flagged in code comments with the reasoning chain. All modulations are testable via the eval framework — behavioral coherence is the validation gate.
**Trade-offs:** Tier 2 modulations extend beyond direct peer-reviewed evidence. Mitigated by grounding in validated dimension-level bridge and eval-framework validation.
**Sources:** Krueger et al. (2012); Gore & Widiger (2013); McCrae & Costa (1989); Blair (2005, 2008, 2013); Bar-Haim et al. (2007); Heydarloo et al. (2024); Kofler et al. (2024); Ekstrom (DSM-III/Jungian mapping)
**Exploration:** deep-analysis
**Status:** captured

## D5: Unify personality pipeline — single path from engram to CDE

**Choice:** Extend Eidos `AgentDisposition` to carry AMPD facets, fix the bridge to read them, deprecate MindMap personality properties. Single path: Engram → Eidos → Bridge → CDE.
**Alternatives:**
- Bridge reads from MindMap instead of Eidos — maintains store split, future LLM sessions will wire into whichever they find first
- Keep both paths active — guaranteed fragmentation, different coding sessions pick different stores
- Fix bridge only (original D2) — patches the symptom without fixing the architecture
**Rationale:** Multiple personality data paths are a trap for LLM-assisted development. MindMap has all AMPD data with clean property names — a future session would naturally build a reader for it, creating a third bridge. Eidos must be the single authoritative personality store. The engram file (extract_personality.py output) is the universal character definition; importing it into Eidos must preserve all personality channels.
**Trade-offs:** Cross-repo change (agency + neocortex). Eidos schema change affects all consumers of AgentDisposition. Mitigated by `ampdFacets` being a new optional field — existing descriptors without it continue to work.
**Sources:** Pipeline trace: PersonalityProfileHandler writes to MindMap, nothing reads back. EidosCognitiveProfileBridge drops mbtiType/enneagramType and passes empty facets.
**Exploration:** deep-analysis
**Depends on:** None — this is the prerequisite for D2, D3, D4
**Status:** captured
