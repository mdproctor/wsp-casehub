## D1: VAD scope — pre-filtering only

**Choice:** Gate: only forward speech chunks to STT, drop silence. SpeechSession unchanged. Endpoint detection deferred.
**Alternatives:**
- Pre-filtering + endpoint detection — also auto-stop when speech ends. Requires SpeechSession changes for always-listening mode. More complex.
- Endpoint detection only — no filtering, all audio reaches STT. Simpler but doesn't reduce unnecessary processing.
**Rationale:** Same compositional injection pattern as denoising (#190). Pre-filtering reduces unnecessary STT processing and avoids transcribing background noise. Endpoint detection is a separate concern that changes SpeechSession behaviour — better as a follow-up.
**Trade-offs:** No auto-stop — user still clicks stop. Acceptable for now; endpoint detection can layer on top later.
**Sources:** sherpa-onnx c-api.h VAD functions, SpeechSession.java (handleAudio/handleStop flow)
**Exploration:** quick
**Status:** captured

## D2: SPI design — separate VoiceActivityFilter interface

**Choice:** Distinct `VoiceActivityFilter` SPI with `filterChunk()` returning samples if speech, empty array if not. Mirrors `StreamingSpeechDenoiser` pattern. Each concern independently testable and toggleable.
**Alternatives:**
- Unified AudioPreprocessor chain — single pipeline abstraction composing denoiser + VAD. Introduces an abstraction for only 2 stages. YAGNI — refactoring cost A→B is low when more stages arrive.
**Rationale:** We have exactly 2 pre-processing stages. The compositional injection pattern is proven. Adding one more optional field to STT services is simpler than a pipeline abstraction. Ordering (denoise → VAD) is implicit but clear at 2 stages.
**Trade-offs:** STT services accumulate optional fields. At 5+ stages, should extract into a pipeline. At 2, the overhead isn't justified.
**Sources:** WhisperSpeechToText.java (existing withStreamingDenoiser pattern), StreamingSpeechDenoiser SPI
**Exploration:** quick
**Status:** captured

## D3: SPI location — speech-api SPI + speech-sherpa implementation

**Choice:** `VoiceActivityFilter` and `VoiceActivityFilterFactory` in `speech-api`, `SherpaOnnxVoiceActivityFilter` in `speech-sherpa`
**Alternatives:**
- speech-sherpa only — no SPI. Couples consumers to sherpa-onnx.
**Rationale:** Same pattern as SpeechDenoiser (D2 from #190). Consistent layering.
**Trade-offs:** None significant — follows established pattern.
**Depends on:** D2 (separate SPI)
**Sources:** SpeechDenoiser.java, StreamingSpeechDenoiser.java (established pattern)
**Exploration:** quick
**Status:** captured

## D4: Model — Silero VAD (silero_vad.onnx)

**Choice:** Silero VAD as the default model. MIT license, 16kHz, well-tested.
**Alternatives:**
- TenVAD — also supported by sherpa-onnx, Apache 2.0 variant. Less widely adopted.
**Rationale:** Silero VAD is the standard choice — widely used, MIT licensed, and the primary VAD model in sherpa-onnx documentation and examples. Default parameters: threshold=0.5, min_silence_duration=0.5s, min_speech_duration=0.25s, window_size=512.
**Trade-offs:** Silero VAD is slightly larger than TenVAD. Negligible for our use case.
**Sources:** sherpa-onnx VAD docs (k2-fsa.github.io/sherpa/onnx/vad/silero-vad.html), Silero VAD repo (github.com/snakers4/silero-vad)
**Exploration:** quick
**Status:** captured

## D5: Runtime configurability — same pattern as denoiser

**Choice:** `casehub.speech.vad.enabled` config property, `BooleanSupplier` injected into STT services. Pass-through when disabled.
**Alternatives:** None — follows established D5 from #190.
**Rationale:** User requirement: all pipeline stages individually toggleable at runtime.
**Trade-offs:** None — same negligible per-call overhead as denoiser toggle.
**Depends on:** D2 (separate SPI with independent toggle)
**Sources:** User direction (session), existing denoiser toggle pattern
**Exploration:** quick
**Status:** captured

## D6: Pipeline ordering — denoise → VAD → STT

**Choice:** Denoiser runs before VAD. VAD operates on clean audio for better speech/silence classification.
**Alternatives:**
- VAD → denoise → STT — VAD on raw audio, only denoise speech. Saves denoising silence but VAD accuracy degrades on noisy audio.
**Rationale:** VAD accuracy improves on clean audio. Denoising silence is cheap (the denoiser processes quickly on quiet input). The quality benefit outweighs the marginal compute saving.
**Trade-offs:** Denoises silence chunks that VAD will then drop. Negligible cost.
**Sources:** DPDFNet/GTCRN performance characteristics, VAD threshold sensitivity
**Exploration:** quick
**Status:** captured
