# HANDOFF — casehub

**Date:** 2026-06-26
**Project:** `/Users/mdproctor/claude/casehub/parent`
**Workspace:** `/Users/mdproctor/claude/public/casehub`

---

## Last Session

**#293 channel taxonomy formalisation — complete. Branch closed, issue closed, blog published.**

- Rewrote `docs/CHANNELS.md` as a 7-section design reference: FIPA 22→9 speech act lineage, 5 purpose categories (12 channel patterns), 8 discriminator dimensions, purpose × semantic matrix, FIPA cross-reference, layered protocol stack, academic lineage.
- Fixed PLATFORM.md: removed EXPIRED from /work (CommitmentState, not MessageType), /observe → EVENT only, removed stale CaseChannelLayout placement violation.
- Fixed qhorus deep-dive: NormativeChannelLayout home updated, /work and /observe types corrected.
- Key design decisions: governance folded into coordination (same speech act pattern); consensus uses APPEND semantic (threshold in backend, not transport); three open patterns sketched (negotiation, consensus, planning).

## Immediate Next Step

Pick next work from the backlog — #294 (Reusable Platform Primitives epic) or qhorus#294 (QhorusCloudEventAdapter timestamp bug).

## What's Left

- `qhorus#294` — bug: QhorusCloudEventAdapter wrong timestamp (affects Drools CEP) · XS · Low

## What's Next

| # | Description | Scale | Complexity | Notes |
|---|-------------|-------|------------|-------|
| #294 | Reusable Platform Primitives epic | XL | High | Long-horizon; channel taxonomy patterns are now ready for implementation issues |

## References

- `docs/CHANNELS.md` — the rewritten taxonomy (authoritative)
- `docs/superpowers/specs/2026-06-25-channel-taxonomy-design.md` — design spec
- `blog/2026-06-26-mdp01-channel-taxonomy-fipa-to-first-principles.md` — diary entry
