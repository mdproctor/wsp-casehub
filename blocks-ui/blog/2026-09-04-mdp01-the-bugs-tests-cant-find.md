---
title: "The bugs tests can't find"
date: 2026-09-04
author: mdp
entry_type: note
subtype: diary
projects: [casehub-blocks-ui]
tags: [speech, testing, tdd, regression, pipeline, kokoro, playwright]
---

108 unit tests. All green. The demo was broken in three different ways.

The speech pipeline queue was drained — ten issues, all complete. Speaker diarization landed, voiceprint identification landed, the campplus embedding extractor, the FFM bindings, everything. Time to test the actual pipeline end-to-end and capture timing baselines. We fired up the avatar demo, selected the Kokoro voice, pressed the mic, spoke a sentence.

The transcript came back scrambled.

## The corrector that corrects wrong

The `TranscriptCorrector` — SymSpell edit-distance spell correction plus phonetic matching — was designed for Zipformer's noisier output. It runs after STT, before the cleanup filters. The problem: it treats clean words as misspelled and "corrects" them into nonsense. "Can you read me a limerick" becomes "An our year new a limerick."

This was fixed once before, in a commit from three days ago — but that fix only removed the corrector from the *typed text* path. The *voice* path still applied it. Same bug, different code path, no test covering the voice path.

The fix is one deleted line. The test is the interesting part: `handleStopDoesNotApplyCorrectorToSttOutput` verifies the corrector is never called on STT output. If someone re-adds it, the test catches it before the demo breaks.

## The voice that wasn't Kokoro

After fixing the corrector, I noticed the Kokoro voices sounded different from what I remembered. Heart had an accent it never had before.

Two bugs stacked on top of each other:

First, the HTML dropdown sent `kokoro:af` as the model key, but the `TtsModelRegistry` maps by `kokoro:af_heart`. No key matched, so every Kokoro selection silently fell back to VITS Lessac — the default. I'd been hearing VITS the entire time and attributing the quality to Kokoro.

Second, the model file was `kokoro-multi-lang-v1_1`. Despite the version number suggesting an improvement over v1.0, v1.1 is a Chinese-specialized model with only three English voices. We'd explicitly chosen v1.0 in a previous session for its 53-voice English focus, but the model name got reverted somewhere. Heart's new accent was real Chinese-optimized phoneme mapping applied to English text.

## What 108 tests missed

Every test used the non-streaming response generator — `null` for the streaming path. But the demo uses the streaming path exclusively. If the streaming branch of `handleStop()` had a bug, no test would catch it. Claude and I added eight tests covering the streaming generator: basic round-trip, timing messages, model selection, multi-sentence TTS, and the full demo configuration with streaming plus speaker identification.

The gap isn't test count. It's test coverage of the code path that actually runs.

## Timing baselines

First systematic capture of end-to-end voice pipeline timings. STT (Zipformer) runs at 34-56ms — a 4-8x improvement over the 184-473ms measured in the September 2 session, likely from the offline recognizer caching work. Cleanup is negligible. LLM round-trip to Vertex AI dominates at 2.1-4.6s depending on response length. TTS (Kokoro warm) adds ~265ms.

Best end-to-end: 2,387ms from mic release to first audio chunk. The pipeline floor is around 2.3 seconds, and almost all of that is waiting for Claude on Vertex.

## The testing gap

Unit tests with mocked dependencies verify that `SpeechSession` calls the right methods in the right order. They don't verify that the dropdown value matches the registry key. They don't verify that the model file is the English version. They don't verify that the corrector makes the transcript worse instead of better.

These are integration-level concerns — the seams between components where assumptions diverge. The dropdown assumes `kokoro:af` is a valid key. The registry knows it isn't. Both are correct in isolation. The bug lives in the gap.

TDD caught the corrector regression because I wrote a test that asserts the corrector is *not* called — a negative assertion about a specific code path. The model version and dropdown values need a different kind of test: one that boots the real CDI container and verifies the wiring end-to-end. That's the next layer to add.
