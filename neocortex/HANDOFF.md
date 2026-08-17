# Handoff — issue-181-embed-separate-regression

**Date:** 2026-07-26
**Branch:** issue-181-embed-separate-regression (closed)
**Status:** Merged to main, pushed to origin + upstream

## What was done

Fixed #181: `embedSeparate()` was producing different retrieval results than
the old `embedBatch()` path, causing a -2.2pp precision regression in the
Hortora engine benchmark.

**Root cause:** The default `embedSeparate` on `MultiModalEmbedder` called
`embed(Map)` which resolved to individual `embed(String)` calls (ONNX batch
size 1). The old retriever code used `embedBatch` (batch size 2). ONNX
transformer models produce subtly different embeddings at different batch
sizes due to padding/attention mask differences.

**Fix:** Changed `embedSeparate` to delegate to `embedBatch` and cherry-pick
dense from index 0, sparse/colbert from index 1 — same batch composition as
the pre-#117 retriever code.

## Files changed

- `inference-api/.../MultiModalEmbedder.java` — `embedSeparate` default method now uses `embedBatch`
- `inference-api/.../MultiModalEmbedderEmbedSeparateTest.java` — new test with `BatchSensitiveEmbedder` verifying batch composition preservation

## Follow-up

- Hortora engine benchmark should re-run with the updated neocortex dependency to confirm precision recovery to 61.6%
- Garden entry GE-20260726-f2a554 captures the ONNX batch-size sensitivity gotcha

## Slot

This work was done in worktree slot 37. The slot can be archived — all work is landed.
