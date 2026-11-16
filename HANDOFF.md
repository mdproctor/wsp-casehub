# HANDOFF — casehub-examples (slot 196)

## Last Session

Completed neocortex#998 (coping emergence eval — Batch 5). 5 LLM eval tests exercising the full CopingMicroLoop with real LLM calls and LLM-as-judge scoring. All tests floor score 4/5. Branch `issue-998-coping-emergence-eval` on neocortex, ready for work-end.

### What Was Built

**#998 — CopingEmergenceEvalTest (branch `issue-998-coping-emergence-eval`):**
- 5 @Tag("llm-eval") tests in `caps-llm-classifier`: conscientiousness→PLANNING, neuroticism→defensive, strategy chaining, reinterpretation coherence, convergence
- Personality profiles designed from predictor mechanics: each forces a specific controllability zone + personality dimension path
- LLM-as-judge scoring with 0–5 scale, retry/backoff, JSON result persistence
- Drive-narrative alignment: drive names/triggers match scenario vocabulary so RelevanceCheck sustains emotions across iterations

### Key Findings

1. **Judge criteria must evaluate behavioural patterns, not strategy names** — SHIFT_RESPONSIBILITY and SUPPRESS_INFORMATION are psychologically equivalent to AVOIDANCE for a neurotic personality. Criteria that name specific enum values score lower than criteria that describe the defensive trajectory.
2. **Strategy chaining is emergent** — CHAINING profile (reinterp=0.70, planful=0.55) at medium controllability (0.45): reinterpretation wins iteration 0 → agency-rich narrative → CopingCheck shifts controllability above 0.6 → predictor filters to INTENTION → planful wins → PLANNING on iteration 1. Cross-category chain from keyword feedback, not scripted.
3. **Drive-narrative alignment is load-bearing** — RelevanceCheck is a keyword matcher. If drive names don't appear in the LLM's natural vocabulary for the scenario, emotions drop below coping threshold and the loop exits after one iteration.

### Code Review

Comprehensive audit: correctness, robustness, consistency, thread safety, dead code. No bugs found. One refactoring opportunity: shared eval infrastructure (extractJson, writeResult, judge) duplicated across CopingEmergenceEvalTest and SleepDerivationEvalTest → filed #1031.

### Blog

`examples/docs/blog/2026-11-16-mdp01-the-loop-that-learns-to-chain.md` — covers predictor mechanics as test harness, keyword feedback chaining, drive-narrative alignment discovery, LLM-as-judge patterns. 2 SVGs.

### Research Docs Updated

- Design spec `wsp-casehub-neocortex/specs/issue-595-coping-strategies/2026-11-15-coping-strategies-design.md` — eval section updated with #998 results table and key findings

### Issues Filed

- #1031 — extract shared eval infrastructure from LLM eval tests (XS/Low)
- #921 — updated with readiness note (coping emergence eval complete, calibration analysis can start)

## Coping Epic (#595) Status

```
Batch 1 (DONE):  #991, #992, #993   types, predictor, LLM generation
Batch 2 (NEXT):  #994, #996         micro-loop orchestrator, goal dispatch + keyword expansion
Batch 3:         #995               regulation refactor
Batch 4:         #997               CAPS + pipeline wiring
Batch 5 (DONE):  #998               emergence eval ← THIS SESSION
Batch 5:         #921               calibration analysis (unblocked, depends on #1031)
Independent:     #999, #1002, #1006, #1007, #1011
New:             #1017, #1018, #1019, #1023, #1031
```

## Neocortex Build Note

Full reactor fails at `schema-generator`. Individual module builds work. For eval tests: `mvn install -pl cognition-api,cognition,cognitive-index -am -Dmaven.test.skip=true` then run eval profile.

## References

- Branch: `issue-998-coping-emergence-eval` on neocortex slot 196
- Design spec: `wsp-casehub-neocortex/specs/issue-595-coping-strategies/2026-11-15-coping-strategies-design.md`
- Blog: `examples/docs/blog/2026-11-16-mdp01-the-loop-that-learns-to-chain.md`
- Eval results: `caps-llm-classifier/target/experiment-results/coping-emergence-eval.json`
