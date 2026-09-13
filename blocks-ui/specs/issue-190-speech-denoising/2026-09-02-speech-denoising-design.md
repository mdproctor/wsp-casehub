# Speech Denoising Pre-processing Design

> **Issue:** casehubio/blocks#190
> **Date:** 2026-09-02
> **Status:** Design
> **Scope:** `speech-api` (SPI) + `speech-sherpa` (FFM implementation) + `speech-ws` (wiring)

## 1. Problem Statement

The speech pipeline transcribes raw audio from microphones in real-world
environments — background noise, echo, fan hum. Without pre-processing, STT
accuracy degrades. sherpa-onnx v1.13.6 ships speech denoising models (GTCRN,
DPDFNet) that clean audio before transcription. This spec adds denoising as
an optional, runtime-configurable pre-processing step.

## 2. SPI Layer (`speech-api`)

### 2.1 Offline Denoiser

```java
package io.casehub.blocks.speech;

public interface SpeechDenoiser {
    float[] denoise(float[] samples, int sampleRate);
}
```

Takes a complete audio buffer, returns denoised samples at the same sample rate.
Used by file-based `SpeechToTextService` implementations.

### 2.2 Streaming Denoiser

```java
package io.casehub.blocks.speech;

public interface StreamingSpeechDenoiser extends AutoCloseable {
    float[] processChunk(float[] samples, int sampleRate);
    void reset();
    void close();
}
```

Processes audio chunks incrementally, maintaining internal state across calls.
`reset()` clears state between utterances. `close()` releases native resources.

### 2.3 Factory

```java
package io.casehub.blocks.speech;

public interface StreamingSpeechDenoiserFactory {
    StreamingSpeechDenoiser create();
}
```

Creates per-stream denoiser instances. Each `RecognitionStream` gets its own
`StreamingSpeechDenoiser` to avoid shared mutable state across concurrent
sessions. The factory itself is thread-safe and holds the cached native engine.

## 3. Implementation (`speech-sherpa`)

### 3.1 SherpaOnnxSpeechDenoiser (offline)

Implements `SpeechDenoiser`. Wraps the offline denoiser FFM bindings.

**Native lifecycle:**
1. `SherpaOnnxCreateOfflineSpeechDenoiser(&config)` — created once, cached
2. `SherpaOnnxOfflineSpeechDenoiserRun(denoiser, samples, n, sampleRate)` — per call
3. `SherpaOnnxDestroyDenoisedAudio(result)` — per call cleanup
4. `SherpaOnnxDestroyOfflineSpeechDenoiser(denoiser)` — on shutdown

**Config struct (`SherpaOnnxOfflineSpeechDenoiserConfig`):**
```
config {
    model {
        gtcrn { model: char* }
        num_threads: int32
        debug: int32
        provider: char*
        dpdfnet { model: char* }
    }
}
```

The FFM layout uses the oversized zero-filled allocation technique
(GE-20260826-190329) — allocate a buffer larger than the struct and zero-fill.
This avoids SIGSEGV from struct layout drift between sherpa-onnx versions
(GE-20260826-51c700).

**Default model:** `dpdfnet_baseline` (2.31M params, 0.36G MACs, 16kHz).

### 3.2 SherpaOnnxStreamingSpeechDenoiser (online)

Implements `StreamingSpeechDenoiserFactory` (factory) and creates
`StreamingSpeechDenoiser` instances (per-stream).

**Native lifecycle:**
1. Factory creates the shared ONNX session config once
2. Per-stream: `SherpaOnnxCreateOnlineSpeechDenoiser(&config)` — one per
   `RecognitionStream`
3. Per-chunk: `SherpaOnnxOnlineSpeechDenoiserRun(denoiser, samples, n, sampleRate)`
   → returns `SherpaOnnxDenoisedAudio*`
4. On stream reset: `SherpaOnnxOnlineSpeechDenoiserReset(denoiser)`
5. On stream close: `SherpaOnnxDestroyOnlineSpeechDenoiser(denoiser)`

**Config struct (`SherpaOnnxOnlineSpeechDenoiserConfig`):**
```
config {
    model {
        gtcrn { model: char* }
        num_threads: int32
        debug: int32
        provider: char*
    }
}
```

**Default model:** `gtcrn_simple` (48.2K params, 33.0 MMACs/sec, 16kHz).
The online API only supports GTCRN — DPDFNet is offline-only.

### 3.3 FFM Bindings

New method handles added to `SherpaLibrary`:

| Handle | C Function |
|--------|-----------|
| `createOfflineDenoiser` | `SherpaOnnxCreateOfflineSpeechDenoiser` |
| `destroyOfflineDenoiser` | `SherpaOnnxDestroyOfflineSpeechDenoiser` |
| `offlineDenoiserRun` | `SherpaOnnxOfflineSpeechDenoiserRun` |
| `destroyDenoisedAudio` | `SherpaOnnxDestroyDenoisedAudio` |
| `createOnlineDenoiser` | `SherpaOnnxCreateOnlineSpeechDenoiser` |
| `destroyOnlineDenoiser` | `SherpaOnnxDestroyOnlineSpeechDenoiser` |
| `onlineDenoiserRun` | `SherpaOnnxOnlineSpeechDenoiserRun` |
| `onlineDenoiserReset` | `SherpaOnnxOnlineSpeechDenoiserReset` |

New layout constants in `SherpaLayouts` for both config structs and
`SherpaOnnxDenoisedAudio` (3 fields: `samples` pointer, `n` int32,
`sample_rate` int32).

## 4. Pipeline Integration

### 4.1 Compositional injection into STT services

STT services gain optional denoiser parameters via `withDenoiser()` /
`withStreamingDenoiser()` builder methods:

**WhisperSpeechToText (streaming, accumulate-then-infer):**
- Accepts `StreamingSpeechDenoiserFactory`
- `startStream()` creates a per-stream `StreamingSpeechDenoiser`
- Each `acceptSamples()` call denoises the chunk before accumulating
- `close()` closes the denoiser alongside the stream

**SherpaOnnxStreamingSpeechToText (streaming, incremental):**
- Same pattern as Whisper — denoises each chunk before feeding the native
  recognizer

**SherpaOnnxSpeechToText (file-based):**
- Accepts `SpeechDenoiser` (offline)
- After reading WAV data, denoises the complete sample buffer before
  building the recognizer config and running inference

### 4.2 Runtime configuration (D5)

All pipeline stages are individually toggleable at runtime:

```properties
casehub.speech.denoising.enabled=true     # default: true when denoiser wired
casehub.speech.denoising.model=dpdfnet_baseline  # offline model selection
casehub.speech.denoising.streaming-model=gtcrn_simple  # online model
```

When `casehub.speech.denoising.enabled=false`, the denoiser passes through
unchanged audio (`processChunk` returns input unchanged, `denoise` returns
input unchanged). The check is per-call, allowing runtime toggle without
restart.

The STT `withDenoiser()` method accepts a `java.util.function.BooleanSupplier`
for the enabled check, bound to the config property:

```java
var stt = WhisperSpeechToText.withDefaults()
    .withStreamingDenoiser(denoiserFactory,
        () -> config.denoisingEnabled());
```

### 4.3 SpeechSession wiring

`SpeechSession` does not change. The denoiser is injected into the STT service
by the CDI producer (`SpeechProducers` in the avatar demo):

```java
@Produces
StreamingSpeechToTextService sttService(DenoiserConfig config) {
    var stt = WhisperSpeechToText.withDefaults();
    if (config.denoisingEnabled()) {
        var factory = SherpaOnnxStreamingSpeechDenoiser.withDefaults();
        stt = stt.withStreamingDenoiser(factory,
            () -> config.denoisingEnabled());
    }
    return stt;
}
```

## 5. Model Provisioning

Add two models to `Provisioner`:

| Model | Release Tag | File | Sample Rate | Params |
|-------|-------------|------|-------------|--------|
| `dpdfnet_baseline` | `speech-enhancement-models` | `dpdfnet_baseline.onnx` | 16kHz | 2.31M |
| `gtcrn_simple` | `speech-enhancement-models` | `gtcrn_simple.onnx` | 16kHz | 48.2K |

Cache locations:
- `~/.casehub/models/sherpa-onnx/dpdfnet_baseline/dpdfnet_baseline.onnx`
- `~/.casehub/models/sherpa-onnx/gtcrn_simple/gtcrn_simple.onnx`

Download URL pattern:
`https://github.com/k2-fsa/sherpa-onnx/releases/download/speech-enhancement-models/<file>`

Add entries to `DENOISER_MODEL_EXPECTED_FILES` map in `Provisioner` (new map,
parallel to `KOKORO_MODEL_EXPECTED_FILES`).

## 6. Testing Strategy

| Test | What It Verifies |
|------|-----------------|
| `SherpaOnnxSpeechDenoiserTest` | Offline: create engine, denoise a known WAV, verify output is non-null, same length, different from input (noise removed) |
| `SherpaOnnxStreamingSpeechDenoiserTest` | Online: create factory, create instance, process chunks, verify per-chunk output, reset between utterances, close lifecycle |
| `WhisperSpeechToTextDenoiserTest` | Integration: STT with denoiser wired produces transcription (mock denoiser verifies it was called) |
| `DenoiserToggleTest` | Runtime toggle: enabled=true calls denoiser, enabled=false passes through unchanged samples |

Tests requiring native libraries (sherpa-onnx) use `@EnabledIf` annotation
gated on native library availability, matching the existing pattern in
`WhisperSpeechToTextTest`.

## 7. Scope Boundary

**In scope:**
- `SpeechDenoiser` and `StreamingSpeechDenoiser` SPIs in `speech-api`
- `SherpaOnnxSpeechDenoiser` (offline) and `SherpaOnnxStreamingSpeechDenoiser`
  (online) in `speech-sherpa`
- FFM bindings for 8 native functions (4 offline, 4 online)
- Model provisioning for `dpdfnet_baseline` and `gtcrn_simple`
- Runtime toggle via config properties
- Integration into `WhisperSpeechToText`, `SherpaOnnxSpeechToText`,
  `SherpaOnnxStreamingSpeechToText`
- Unit tests with mocked denoiser, integration tests gated on native libs

**Out of scope:**
- Benchmarking different DPDFNet model variants (dpdfnet2/4/8)
- Custom attenuation limit configuration (use sherpa-onnx defaults)
- Demo app UI controls for toggling denoising (future enhancement)
- Noise level detection / adaptive denoising

## References

- [sherpa-onnx C API header](https://github.com/k2-fsa/sherpa-onnx/blob/master/sherpa-onnx/c-api/c-api.h)
- [DPDFNet speech enhancement docs](https://k2-fsa.github.io/sherpa/onnx/speech-enhancement/dpdfnet.html)
- [DPDFNet C API example](https://github.com/k2-fsa/sherpa-onnx/blob/master/c-api-examples/speech-enhancement-dpdfnet-c-api.c)
- [GTCRN official repo](https://github.com/Xiaobin-Rong/gtcrn)
- [sherpa-onnx speech-enhancement-models release](https://github.com/k2-fsa/sherpa-onnx/releases/tag/speech-enhancement-models)
- GE-20260826-51c700 — sherpa-onnx FFM struct layout must match ALL nested sub-configs
- GE-20260826-190329 — Oversized zero-filled allocation technique for version-resilient FFM
- GE-20260803-e363e6 — ONNX Runtime SIGSEGV with concurrent thread pool access
- `speech-api/src/main/java/io/casehub/blocks/speech/SpeechToTextService.java`
- `speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechToTextService.java`
- `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java`
- `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechToText.java`
- `speech-ws/src/main/java/io/casehub/blocks/speech/ws/SpeechSession.java`
