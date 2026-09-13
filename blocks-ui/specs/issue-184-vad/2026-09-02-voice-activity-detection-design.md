# Voice Activity Detection (VAD) Pre-filtering Design

> **Issue:** casehubio/blocks#184
> **Date:** 2026-09-02
> **Status:** Design
> **Scope:** `speech-api` (SPI) + `speech-sherpa` (FFM implementation)

## 1. Problem Statement

The speech pipeline feeds all audio to STT — including silence, background
noise, and non-speech sounds. This wastes STT compute and can produce
phantom transcriptions from noise. VAD classifies each audio chunk as
speech or silence, gating non-speech chunks before they reach the STT
engine.

## 2. SPI Layer (`speech-api`)

### 2.1 VoiceActivityFilter

```java
package io.casehub.blocks.speech;

public interface VoiceActivityFilter extends AutoCloseable {
    float[] filterChunk(float[] samples, int sampleRate);
    void reset();
    void close();
}
```

`filterChunk` returns the input samples unchanged when speech is detected,
or an empty `float[0]` when silence is detected. The caller skips buffer
accumulation when the result is empty.

`reset()` clears internal state between utterances. `close()` releases
native resources.

### 2.2 Factory

```java
package io.casehub.blocks.speech;

public interface VoiceActivityFilterFactory {
    VoiceActivityFilter create();
}
```

Each `RecognitionStream` gets its own `VoiceActivityFilter` to avoid
shared mutable state across concurrent sessions. The factory holds the
model config; per-stream instances hold the native VAD handle.

## 3. Implementation (`speech-sherpa`)

### 3.1 SherpaOnnxVoiceActivityFilter

Implements `VoiceActivityFilterFactory` (factory) and creates
`VoiceActivityFilter` instances (per-stream).

**Native lifecycle:**
1. Factory creates config (model path, threshold, timing params)
2. Per-stream: `SherpaOnnxCreateVoiceActivityDetector(&config, bufferSizeSeconds)`
3. Per-chunk: `SherpaOnnxVoiceActivityDetectorAcceptWaveform(vad, samples, n)`,
   then check `SherpaOnnxVoiceActivityDetectorDetected(vad)` — if true, return
   samples; if false, return empty array
4. On stream reset: `SherpaOnnxVoiceActivityDetectorReset(vad)`
5. On stream close: `SherpaOnnxDestroyVoiceActivityDetector(vad)`

**Default model:** Silero VAD (`silero_vad.onnx`, MIT license, 16kHz).

**Default parameters:**
- `threshold`: 0.5 (speech probability threshold)
- `min_silence_duration`: 0.5s
- `min_speech_duration`: 0.25s
- `window_size`: 512 samples
- `max_speech_duration`: 20.0s
- `buffer_size_in_seconds`: 30.0

### 3.2 FFM Bindings

New method handles added to `SherpaLibrary`:

| Handle | C Function |
|--------|-----------|
| `createVad` | `SherpaOnnxCreateVoiceActivityDetector` |
| `destroyVad` | `SherpaOnnxDestroyVoiceActivityDetector` |
| `vadAcceptWaveform` | `SherpaOnnxVoiceActivityDetectorAcceptWaveform` |
| `vadDetected` | `SherpaOnnxVoiceActivityDetectorDetected` |
| `vadReset` | `SherpaOnnxVoiceActivityDetectorReset` |
| `vadFlush` | `SherpaOnnxVoiceActivityDetectorFlush` |

Note: `Front`, `Pop`, `Empty`, `Clear`, `DestroySpeechSegment` are not
needed for the pre-filtering use case — we only need `Detected()` to
check the current speech state. Segment extraction is for the endpoint
detection use case (deferred).

**Config struct (`SherpaOnnxVadModelConfig`):**
```
config {
    silero_vad {
        model: char*           // offset 0
        threshold: float       // offset 8
        min_silence_duration: float  // offset 12
        min_speech_duration: float   // offset 16
        window_size: int32     // offset 20
        max_speech_duration: float   // offset 24
    }                          // 28 bytes + padding = 32
    sample_rate: int32         // offset 32
    num_threads: int32         // offset 36
    provider: char*            // offset 40
    debug: int32               // offset 48
    ten_vad { ... }            // offset 52+ (unused, zeroed)
}
```

Uses the oversized zero-filled allocation (4096 bytes) per
GE-20260826-190329.

## 4. Pipeline Integration

### 4.1 Compositional injection

STT services gain `withVoiceActivityFilter()` builder methods, following
the same pattern as `withStreamingDenoiser()`:

```java
var stt = WhisperSpeechToText.withDefaults()
    .withStreamingDenoiser(denoiserFactory, () -> config.denoisingEnabled())
    .withVoiceActivityFilter(vadFactory, () -> config.vadEnabled());
```

Inside `acceptSamples()`, after denoising and before buffer accumulation:

```java
float[] processed = samples;
// step 1: denoise (existing)
if (denoiser != null && denoiserEnabled.getAsBoolean()) {
    processed = denoiser.processChunk(samples, sampleRate);
}
// step 2: VAD gate (new)
if (vadFilter != null && vadEnabled.getAsBoolean()) {
    processed = vadFilter.filterChunk(processed, sampleRate);
}
// step 3: accumulate (only if non-empty)
if (processed.length == 0) { return; }
// ... existing buffer logic with 'processed' ...
```

### 4.2 Runtime configuration

```properties
casehub.speech.vad.enabled=true
```

Same `BooleanSupplier` pattern as the denoiser toggle. Per-call check
allows runtime toggle without restart.

### 4.3 SpeechSession wiring

`SpeechSession` does not change. The VAD filter is injected into the STT
service by `SpeechProducers` in the avatar demo, alongside the denoiser:

```java
var stt = whisper.withStreamingDenoiser(denoiserFactory, denoisingEnabled::get)
                 .withVoiceActivityFilter(vadFactory, vadEnabled::get);
```

## 5. Model Provisioning

Add Silero VAD to `Provisioner`:

| Model | Source | File | Sample Rate |
|-------|--------|------|-------------|
| `silero_vad` | sherpa-onnx releases | `silero_vad.onnx` | 16kHz |

Download URL: `https://github.com/k2-fsa/sherpa-onnx/releases/download/vad-models/silero_vad.onnx`

Cache location: `~/.casehub/models/sherpa-onnx/silero_vad/silero_vad.onnx`

Add to `VAD_MODEL_EXPECTED_FILES` map in `Provisioner`.

## 6. Testing Strategy

| Test | What It Verifies |
|------|-----------------|
| `SherpaOnnxVoiceActivityFilterTest` | Factory creates instances, filterChunk returns non-empty for speech audio, empty for silence, reset clears state, close lifecycle |
| `VadIntegrationTest` | STT with VAD wired: mock VAD verifies filterChunk called, runtime toggle works (enabled=true calls filter, false passes through) |

Tests requiring native libraries gated with `@EnabledIf` on library
availability.

## 7. Scope Boundary

**In scope:**
- `VoiceActivityFilter` and `VoiceActivityFilterFactory` SPIs in `speech-api`
- `SherpaOnnxVoiceActivityFilter` in `speech-sherpa`
- FFM bindings for 6 native functions
- Model provisioning for `silero_vad`
- Runtime toggle via config property
- Integration into `WhisperSpeechToText`, `SherpaOnnxStreamingSpeechToText`
- Pipeline ordering: denoise → VAD → STT

**Out of scope:**
- Endpoint detection (auto-stop when speech ends) — separate issue
- Always-listening mode (requires SpeechSession changes)
- File-based SpeechToTextService VAD integration (VAD is streaming-only)
- Segment extraction (`Front`/`Pop` API)
- Custom VAD threshold configuration UI

## References

- [sherpa-onnx C API header](https://github.com/k2-fsa/sherpa-onnx/blob/master/sherpa-onnx/c-api/c-api.h) — VAD struct and function declarations
- [Silero VAD docs](https://k2-fsa.github.io/sherpa/onnx/vad/silero-vad.html) — model parameters and usage
- [Silero VAD repo](https://github.com/snakers4/silero-vad) — model source, MIT license
- GE-20260826-51c700 — sherpa-onnx FFM struct layout gotcha
- GE-20260826-190329 — oversized zero-filled allocation technique
- `speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiser.java` — pattern reference
- `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java` — integration point
- `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechToText.java` — integration point
- `speech-ws/src/main/java/io/casehub/blocks/speech/ws/SpeechSession.java` — unchanged by this spec
- 2026-09-02-speech-denoising-design.md — denoiser spec (establishes the pattern)
