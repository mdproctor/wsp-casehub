---
layout: post
title: "The Audio Pipeline Nobody Documents"
date: 2026-09-02
entry_type: article
subtype: diary
projects: [casehubio/blocks-ui, casehubio/blocks]
tags: [speech, sherpa-onnx, denoising, vad, ffm, pipeline, audio]
series: issue-190-speech-denoising
---

# The Audio Pipeline Nobody Documents

sherpa-onnx ships speech denoising and voice activity detection alongside its STT and TTS models, but the documentation assumes you're calling a command-line tool. If you're embedding these via Java FFM — calling the C API from a Java 22+ process without JNI — you're on your own for the struct layouts, lifecycle management, and pipeline composition. This is the reference I wished existed when I started.

## The pipeline

Raw microphone audio is noisy. Background hum, keyboard clicks, fan noise. Feeding it directly to STT produces phantom transcriptions — words that aren't there. Two pre-processing stages fix this:

```
audio chunks → denoiser → VAD → STT
```

**Denoiser** cleans the audio — removes background noise, returns cleaned samples at the same sample rate. The audio shape doesn't change; the content does.

**VAD** (Voice Activity Detection) gates the audio — classifies each chunk as speech or silence. Speech chunks pass through unchanged. Silence chunks are dropped entirely (empty array). STT never sees them.

The ordering matters. Denoise first, then VAD. VAD accuracy improves on clean audio — running it on raw noisy input causes false negatives (missed speech buried in noise). The cost of denoising silence chunks that VAD then drops is negligible.

## sherpa-onnx model selection

sherpa-onnx offers multiple models for each stage. The choice depends on whether you're processing files (offline) or streaming chunks from a microphone (online).

### Denoising

| Model | Params | MACs | Sample Rate | API |
|-------|--------|------|-------------|-----|
| dpdfnet_baseline | 2.31M | 0.36G | 16kHz | Offline only |
| dpdfnet2 | 2.49M | 1.35G | 16kHz | Offline only |
| dpdfnet4 | 2.84M | 2.36G | 16kHz | Offline only |
| dpdfnet8 | 3.54M | 4.37G | 16kHz | Offline only |
| gtcrn_simple | 48.2K | 33.0M | 16kHz | Online (streaming) |

DPDFNet is the higher-quality family but only available for offline processing — you need the complete audio buffer. GTCRN is orders of magnitude smaller (48K vs 2.3M parameters) and supports streaming — it processes chunks as they arrive.

For a real-time avatar pipeline where audio arrives as WebSocket frames, GTCRN is the only option. For file-based transcription, DPDFNet baseline gives the best quality-to-compute ratio.

The critical discovery: **the online streaming denoiser only supports GTCRN**. The offline denoiser supports both GTCRN and DPDFNet. This isn't documented clearly — I found it by reading the C API header directly. The online config struct has no `dpdfnet` sub-struct.

### VAD

Silero VAD is the standard choice — MIT licensed, 16kHz, widely tested. The key parameters:

- `threshold`: 0.5 — speech probability above this = speech detected
- `min_silence_duration`: 0.5s — silence shorter than this doesn't close a speech segment
- `min_speech_duration`: 0.25s — speech shorter than this is ignored
- `window_size`: 512 samples (32ms at 16kHz) — the classification granularity

The `min_silence_duration` is load-bearing. Set it too low and the VAD chops words apart on brief pauses. Set it too high and it takes half a second of silence before the gate closes. 0.5s is the right default for conversational speech.

## FFM integration — the struct layout trap

Every sherpa-onnx C API call follows the same pattern: allocate a config struct, zero-fill it, set the fields you care about, call the create function, use the handle, destroy it. The Java FFM equivalent:

```java
MemorySegment config = arena.allocate(4096);
config.fill((byte) 0);
config.set(ValueLayout.ADDRESS, OFFSET, arena.allocateFrom(modelPath));
config.set(ValueLayout.JAVA_INT, OFFSET + 8, numThreads);
```

The 4096-byte allocation is deliberate. sherpa-onnx config structs embed sub-configs for every supported model type — even the ones you're not using. The offline recognizer config has **17** nested model sub-structs. If you allocate only enough for the fields you set, the C library reads past your allocation into uninitialised memory and SIGSEGVs.

The zero-fill is the fix. sherpa-onnx treats zero/NULL fields as "not configured" — same as C's `memset(&config, 0, sizeof(config))` pattern. Allocating more than the struct needs is harmless; allocating less crashes.

The `SherpaOnnxDenoisedAudio` result struct is identical in layout to `SherpaOnnxGeneratedAudio` from the TTS API — both are `{float* samples, int32 n, int32 sample_rate}`. The destroy functions are different (`SherpaOnnxDestroyDenoisedAudio` vs `SherpaOnnxDestroyOfflineTtsGeneratedAudio`), but the VarHandles for reading the result fields are reusable.

## Composable pipeline design

The denoiser and VAD are separate SPIs with separate runtime toggles. Each STT service accepts them optionally via builder methods:

```java
var stt = WhisperSpeechToText.withDefaults()
    .withStreamingDenoiser(denoiserFactory, () -> config.denoisingEnabled())
    .withVoiceActivityFilter(vadFactory, () -> config.vadEnabled());
```

The `BooleanSupplier` for each toggle checks a config property per call — not at construction time. Flip `casehub.speech.denoising.enabled` from `true` to `false` at runtime and the denoiser passes through on the next chunk, no restart needed. Same for VAD.

Inside `acceptSamples()`, the pipeline is three lines:

```java
float[] processed = samples;
if (denoiser != null && denoiserEnabled.getAsBoolean())
    processed = denoiser.processChunk(samples, sampleRate);
if (vadFilter != null && vadEnabled.getAsBoolean())
    processed = vadFilter.filterChunk(processed, sampleRate);
if (processed.length == 0) return;
// ... buffer accumulation with 'processed' ...
```

Each stage transforms or gates independently. The VAD returning an empty array short-circuits buffer accumulation — the STT engine never sees the silence.

I considered a unified `AudioPreprocessor` pipeline abstraction to compose stages, but with only two stages, three similar lines is simpler than a premature abstraction. If we reach four or five stages (AGC, echo cancellation, resampling), the extraction cost is low and the need will be obvious.

## The factory pattern for streaming state

Both the streaming denoiser and VAD maintain internal state across chunks — the GTCRN model tracks temporal context, the VAD tracks speech/silence transitions. Each concurrent WebSocket session needs its own instance.

The factory pattern handles this cleanly: the factory holds the model config (shared, thread-safe), and each `RecognitionStream` creates its own denoiser/VAD instance via `factory.create()`. The instance lives for the duration of one recording session and is closed with the stream.

```java
// Factory: created once at startup, shared across sessions
StreamingSpeechDenoiserFactory factory = SherpaOnnxStreamingSpeechDenoiser.withDefaults();

// Per-stream: created in startStream(), closed in stream.close()
StreamingSpeechDenoiser denoiser = factory.create();
```

The online VAD has a `reset()` method for clearing state between utterances within the same session, but for our push-to-talk model, each recording creates a fresh stream and a fresh denoiser/VAD instance. Reset is there for always-listening scenarios where the same VAD instance processes multiple utterances — a future enhancement.

## What this opens up

The composable pipeline makes each stage independently measurable. With runtime toggles, A/B comparison is trivial: run the same utterance with denoising on and off, compare transcription accuracy. Same for VAD — does dropping silence chunks actually improve Whisper's output, or does Whisper handle silence fine on its own?

The VAD `Detected()` state also enables endpoint detection — knowing when the user stopped speaking without requiring them to click a button. That's a separate concern from pre-filtering (the VAD gates chunks; endpoint detection triggers `finalResult()`), but the same native handle provides both signals. The pre-filtering gate is in; endpoint detection is the natural next step.
