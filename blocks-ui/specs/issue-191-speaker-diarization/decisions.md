## D1: Scope — full pipeline with merged #211

**Choice:** Full pipeline covering offline diarization + real-time speaker identification + speaker-attributed transcription, with #211 merged into this issue.
**Alternatives:**
- Offline diarization only (reviewer revision) — technically cleaner boundary but delivers the secondary feature first. The user's primary use case (avatar recognises family members) requires real-time speaker ID from #211. Splitting creates coordination overhead since both share the embedding model, provisioning, and SPI layer.
- Diarization engine only — produces segments without transcription. Defers value.
**Rationale:** "Speaker awareness" is one capability from the user's perspective. The shared foundation (campplus model, provisioning, SPI design) means splitting creates rework at the seam. Both features target the same branch and developer — no benefit from independent delivery.
**Trade-offs:** XL/High scope, but decomposes into independent batches (shared foundation → offline diarization → real-time speaker ID + avatar integration).
**Sources:** casehubio/blocks#191, casehubio/blocks#211, user: "think about how a family interacts"
**Exploration:** quick → reviewer revised → user override (first-principles analysis: shared foundation, single developer, primary use case requires both)
**Status:** captured (user override of reviewer revision)

## D2: Mode — both real-time and offline

**Choice:** Both real-time speaker identification (during live avatar conversations) and offline diarization (for recorded audio).
**Alternatives:**
- Offline only (reviewer revision) — technically separate problems (batch clustering vs per-turn matching) but the embedding extractor and model provisioning are shared.
- Real-time only — misses meeting transcription use case.
**Rationale:** Real-time speaker ID is the primary use case. Offline diarization is also needed. Different inference paths (sherpa-onnx C API for offline, ORT for real-time) are an implementation detail addressed in the architecture, not a reason to split.
**Trade-offs:** More implementation surface than either mode alone.
**Sources:** User requirement: "think about how a family interacts, it would be good if it automatically recognised which family member it's talking to"
**Exploration:** quick → reviewer revised → user override
**Status:** captured (user override of reviewer revision)

## D3: Enrollment — hybrid (auto-detect + explicit)

**Choice:** Hybrid enrollment: auto-detect unknown speakers and prompt for name, plus an explicit enrollment option for better initial accuracy.
**Alternatives:**
- Auto-enroll only — zero friction but lower initial voiceprint quality.
- Explicit enrollment only — higher accuracy but breaks conversation flow.
- Deferred to #211 (reviewer revision) — no longer applicable since #211 is merged.
**Rationale:** Auto-detect provides zero-friction onboarding for families. Explicit enrollment option gives better initial accuracy when desired. Both paths feed the same voiceprint registry.
**Trade-offs:** Two enrollment paths to implement and test. Reviewer flagged minimum audio duration and detection thresholds as design concerns — addressed in the spec.
**Sources:** Avatar demo interaction flow (SpeechSession.java, SpeechWebSocket.java), R1-08
**Exploration:** quick
**Status:** captured (restored from user override of D1)

## D4: Voiceprint persistence — dual (local + server-side)

**Choice:** SPI-abstracted voiceprint store with both file-system and REST API implementations.
**Alternatives:**
- In-memory only (reviewer Tier 1 suggestion) — reasonable for initial development, but the SPI abstraction costs nothing and file persistence is trivial to add.
- Local file system only — simple but no multi-device access.
- Server-side only — adds server dependency, privacy concerns for biometric data.
- Deferred to #211 (reviewer revision) — no longer applicable.
**Rationale:** Local storage works offline and fits the local-first sherpa-onnx model. Server-side enables multi-device access in production. SPI abstraction keeps the core engine agnostic to storage backend. Reviewer's privacy concerns (GDPR Article 9, BIPA, encryption at rest) are valid and addressed in the spec.
**Trade-offs:** Two storage implementations to maintain. Privacy considerations for biometric data.
**Sources:** Existing pattern: Provisioner caches to ~/.casehub/, platform APIs for production, R1-07
**Exploration:** quick
**Status:** captured (restored from user override of D1)

## D5: Embedding architecture — dual inference paths

**Choice:** sherpa-onnx C API (`SherpaOnnxCreateOfflineSpeakerDiarization`) for offline diarization. ORT with campplus.onnx (`OnnxRuntimeLibrary`) for real-time speaker embedding extraction, following the proven path in `CosyVoice3VoiceEncoder.extractSpeakerEmbedding`.
**Alternatives:**
- sherpa-onnx C API for everything (original D5) — binds additional C functions for speaker embedding extraction when ORT is already proven for campplus.
- Pure Java/ORT for everything — reimplements sherpa-onnx's diarization pipeline (segmentation, overlap handling, spectral clustering).
**Rationale:** Offline diarization is sherpa-onnx's strength — the C API bundles segmentation, overlap handling, and spectral clustering. For real-time embedding extraction, ORT is already proven: `CosyVoice3VoiceEncoder.extractSpeakerEmbedding` extracts campplus embeddings via `OnnxRuntimeLibrary.Session.runFloat()` with established preprocessing. Each mode uses its best tool. Both use campplus embeddings — the model is shared even though inference paths differ.
**Trade-offs:** Offline diarization's internal embeddings and ORT-extracted embeddings are both campplus 192-dim, but offline discovers speakers via internal clustering while real-time compares against enrolled voiceprints — they don't need to interoperate.
**Depends on:** D8 (campplus model choice)
**Sources:** SherpaLibrary.java, CosyVoice3VoiceEncoder.extractSpeakerEmbedding, R1-01 (reviewer's insight about dual paths — adopted)
**Exploration:** quick
**Status:** captured (incorporates reviewer's dual-path insight)

## D6: SPI shape — layered, 3 composable interfaces

**Choice:** Three independent interfaces in `speech-api`:
- `SpeakerEmbeddingExtractor`: `SpeakerEmbedding extract(float[] samples, int sampleRate)` — real-time embedding extraction for speaker ID.
- `SpeakerRegistry`: `register(String name, SpeakerEmbedding)`, `identify(SpeakerEmbedding) → SpeakerMatch(name, confidence)` — with pluggable `VoiceprintStore` SPI for persistence.
- `SpeakerDiarizationService`: `List<DiarizedSegment> diarize(Path audioFile, DiarizationOptions options)` — offline diarization. `DiarizedSegment(long startMs, long endMs, String speakerLabel, float[] samples, int sampleRate)` includes extracted audio samples so consumers can compose with STT without re-reading the original file.
**Alternatives:**
- Single SpeakerDiarizationService only (reviewer revision for offline-only scope) — insufficient for merged scope.
- Unified SpeakerService — conflates concerns.
- Combined diarize-and-transcribe method — couples diarization and transcription SPIs, violating platform's composable SPI pattern.
**Rationale:** Each interface serves a distinct consumer. Real-time speaker ID composes Extractor + Registry. Offline diarization is self-contained. The embedding extractor is NOT reusable by CosyVoice3 (its internal SpeakerExtractor takes preprocessed `float[][] logMel`, not raw audio) — the SPI serves speaker ID, not voice cloning.
**Trade-offs:** Three interfaces to implement, but each is small and focused.
**Depends on:** D5 (dual inference paths determine implementations)
**Sources:** speech-api SPI pattern, R1-03 (CosyVoice3 incompatibility — adopted), R2-01 (audio samples in DiarizedSegment — adopted), R2-02 (Path-based input — adopted)
**Exploration:** quick
**Status:** captured (incorporates reviewer's SPI improvements while restoring 3-interface design)

## D7: Avatar integration — post-STT, per turn

**Choice:** Speaker identification runs in parallel with STT on the same audio buffer, per conversation turn. The speaker label is passed to the LLM as context ("You are talking to Mark").
**Alternatives:**
- Pre-STT gate — adds latency to the critical path.
- Deferred to #211 (reviewer revision) — no longer applicable since #211 is merged.
**Rationale:** Embedding extraction takes ~50ms on a few seconds of speech; STT takes 200-500ms+. Running in parallel adds zero latency. The speaker label enriches LLM context for personalised responses. Requires structural changes to SpeechSession (accumulation buffer), ConversationTurn (speaker field), PromptAssembler (speaker parameter).
**Trade-offs:** Speaker ID result must be available before the LLM call — if extraction fails or times out, the turn proceeds without a speaker label.
**Depends on:** D6 (SPI shape — avatar composes Extractor + Registry)
**Sources:** SpeechSession.java, ConversationTurn.java, PromptAssembler.java, R1-04 (structural change analysis — adopted)
**Exploration:** quick
**Status:** captured (restored from user override of D1, incorporates reviewer's structural analysis)

## D8: Model architecture — campplus (explicit choice)

**Choice:** campplus as the default speaker embedding model for both offline diarization and real-time speaker ID.
**Alternatives:**
- ECAPA-TDNN (SpeechBrain) — strong VoxCeleb benchmark performance, ~192-dim, ~20MB. Upgrade path if campplus accuracy is insufficient.
- WeSpeaker / 3D-Speaker — potentially better multilingual support, supported by sherpa-onnx.
- TitaNet (NVIDIA) — state-of-the-art accuracy, available as ONNX export.
**Rationale:** campplus is already provisioned and loaded for CosyVoice3 voice cloning (campplus.onnx, 192-dim embeddings). Using the same model avoids provisioning additional artifacts. Adequate for the family interaction use case. ECAPA-TDNN is the recommended upgrade path if accuracy demands it.
**Trade-offs:** campplus is not the highest-accuracy option. Family use case (2-8 speakers, familiar voices) has modest discrimination requirements.
**Sources:** campplus.onnx (Provisioner.java), CosyVoice3VoiceEncoder (192-dim, CAMPPLUS_MEL config), R1-06
**Exploration:** quick (surfaced by reviewer)
**Status:** captured

## D9: Module placement — speech-api SPI, speech-sherpa implementation

**Choice:** All three SPI interfaces (`SpeakerEmbeddingExtractor`, `SpeakerRegistry`, `SpeakerDiarizationService`) in `speech-api` module (zero foundation dependencies). sherpa-onnx diarization and ORT embedding implementations in `speech-sherpa` module. `VoiceprintStore` SPI in `speech-api`, file-system implementation in `speech-sherpa`, REST implementation in `speech-sherpa` or a separate module.
**Alternatives:**
- All in speech-sherpa — loses provider-agnostic SPI abstraction.
- New dedicated module — unnecessary; established two-module pattern handles this.
**Rationale:** Following the established pattern: pure SPI interfaces → `speech-api`, implementations → `speech-sherpa`. All three interfaces add no internal dependencies to `speech-api` — they use pure Java records as return types.
**Sources:** speech-api module, speech-sherpa module, R1-10
**Exploration:** quick (surfaced by reviewer)
**Status:** captured
