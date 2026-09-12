## D1: Denoiser scope — both offline and online (streaming)

**Choice:** Implement both offline denoiser (for file-based STT) and online streaming denoiser (for WebSocket/avatar path)
**Alternatives:**
- Offline only — matches issue scope exactly but leaves the primary consumer (avatar) without denoising
- Online only — skips file-based path, incomplete
**Rationale:** The avatar WebSocket path is the primary consumer and processes audio chunk-by-chunk. The offline denoiser can't work there without buffering all audio first (breaking partial results). sherpa-onnx provides both APIs (offline and online streaming with chunk-duration-ms). Implementing both covers all paths in one pass.
**Trade-offs:** Two FFM binding sets instead of one. Mitigated by shared model provisioning and similar struct layouts.
**Sources:** sherpa-onnx c-api.h, DPDFNet docs (k2-fsa.github.io/sherpa/onnx/speech-enhancement/dpdfnet.html), SpeechSession.java, WhisperSpeechToText.java
**Exploration:** quick
**Status:** captured

## D2: SPI location — speech-api SPI + speech-sherpa implementation

**Choice:** `SpeechDenoiser` interface in `speech-api`, `SherpaOnnxSpeechDenoiser` implementation in `speech-sherpa`
**Alternatives:**
- speech-sherpa only — no SPI, concrete class only. Simpler but couples consumers to sherpa-onnx
**Rationale:** Follows the existing pattern exactly. `SpeechToTextService` and `StreamingSpeechToTextService` are SPIs in `speech-api` with implementations in `speech-sherpa`. The denoiser should follow the same layering.
**Trade-offs:** Slightly more code (interface + impl vs just impl). Worth it for consistency and future flexibility.
**Sources:** speech-api/src/main/java/io/casehub/blocks/speech/SpeechToTextService.java, speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechToTextService.java
**Exploration:** quick
**Status:** captured

## D3: Pipeline integration — compositional injection into STT services

**Choice:** STT services accept an optional `SpeechDenoiser` via builder/factory method. Denoising happens transparently inside the STT service — callers unchanged.
**Alternatives:**
- Decorator wrapping STT service — clean separation but breaks partial results for offline denoiser path (buffers everything until finalResult)
- Explicit step in SpeechSession — simple but couples denoising to WebSocket layer, file-based path needs separate wiring
**Rationale:** Compositional injection keeps denoising transparent to all callers. For the online denoiser: each chunk is denoised before feeding to the recognition stream. For the offline denoiser in file-based mode: complete samples denoised before recognition. SpeechSession doesn't change.
**Trade-offs:** STT service constructors grow by one optional parameter. Acceptable — it's the same pattern as vocabularyHint and other optional features.
**Depends on:** D1 (both paths), D2 (SPI in speech-api)
**Sources:** SpeechSession.java (handleAudio/handleStop flow), WhisperSpeechToText.java (accumulate-then-infer pattern)
**Exploration:** quick
**Status:** captured

## D4: Default model — dpdfnet_baseline (16kHz)

**Choice:** `dpdfnet_baseline` (2.31M params, 0.36G MACs, 16kHz) as the default model
**Alternatives:**
- dpdfnet2 (2.49M, 1.35G MACs) — better quality, ~4x more compute
- dpdfnet4 (2.84M, 2.36G MACs) — higher quality, ~6.5x more compute
**Rationale:** Denoising adds latency to every utterance in the avatar pipeline. The baseline model is the fastest with lowest resource usage. Since the ONNX session is created once and reused (per #225 caching pattern), the model load time is amortised. Quality is sufficient for noise suppression before STT.
**Trade-offs:** May miss subtle noise that dpdfnet2/4/8 would catch. Configurable at runtime (D5), so users can switch models without code changes.
**Sources:** DPDFNet docs (k2-fsa.github.io), dpdfnet_baseline.onnx on sherpa-onnx releases
**Exploration:** quick
**Status:** captured

## D5: Runtime configurability — all pipeline stages toggleable

**Choice:** Every pipeline stage (denoising, correction, cleanup) is individually toggleable at runtime via configuration, defaulting to pass-through when disabled
**Alternatives:**
- Compile-time only — simpler but prevents A/B comparison and debugging
- Per-session toggle — WebSocket message to enable/disable stages; more flexible but complex
**Rationale:** User requirement: "anything added to the pipeline should be configurable to pass through via runtime configuration" for easy evaluation and debugging. Configuration via system properties or Quarkus config properties allows toggling without restart.
**Trade-offs:** Slight per-call overhead checking config flags. Negligible compared to the denoising/inference cost itself.
**Sources:** User direction (session)
**Exploration:** quick
**Status:** captured
