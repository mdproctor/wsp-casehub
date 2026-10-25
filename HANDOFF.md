# HANDOFF — casehub-examples (slot 196)

## Last Session

Cognitive research consolidation arc complete (#144-#148, all closed). Set up implementation queue for cognitive pipeline work.

## Immediate Next Step

`work start neocortex#655` — first issue in the queue.

## Implementation Path (10 issues, 5 batches)

### Batch 1: SessionPipeline migration (neocortex#657 remaining)
| # | Repo | Title | Scale |
|---|------|-------|-------|
| 655 | neocortex | Remove remaining AgentProvider usage (CommunitySummaryPhase, LlmReflectionSynthesizer) | S |
| 658 | neocortex | Generic stub SessionPipeline for testing | S |
| 352 | blocks | Fix BlocksAutoConfiguration for SessionPipeline migration | S |

### Batch 2: Scenario validation gate
| # | Repo | Title | Scale |
|---|------|-------|-------|
| 135 | examples | Full scenario run with all cognitive components enabled | M |

### Batch 3: Drive expression pipeline (examples#122 Batch 1 — get working)
| # | Repo | Title | Scale |
|---|------|-------|-------|
| 685 | neocortex | Import caps_patterns — seed CAPS weights from biography | M |
| 686 | neocortex | Import behavioural_attractors as BEHAVIORAL subgraph nodes | M |
| 122 | examples | Drive expression pipeline — get working + multi-channel | L |

### Batch 4: Multi-channel expression (examples#122 Batch 2)
| # | Repo | Title | Scale |
|---|------|-------|-------|
| 668 | neocortex | Situation-dependent channel weighting | M |
| 607 | neocortex | Frijda action tendencies literature | S |

### Batch 5: Tuning
| # | Repo | Title | Scale |
|---|------|-------|-------|
| 102 | examples | Cognitive pipeline tuning — graduated nodes, attractor strength, per-character differentiation | M |

## Cross-Module

- Batch 1 spans neocortex + blocks
- Batch 3 spans neocortex + examples
- All batches are sequential — each depends on the prior batch

## References

- `.plan` — queue with 10 issues in dependency order
- `cognitive-research-inventory.md` — 57-document audit (context for implementation)
- `neocortex/docs/research/` — era1, era2, era3 journals (the research behind the implementation)
- `JOURNAL.md` — session journal
