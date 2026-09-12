# Speaker Diarization & Identification — Design Spec

**Issue:** casehubio/blocks#191 (Speaker diarization), casehubio/blocks#211 (Speaker identification/voiceprint) — merged
**Date:** 2026-09-03
**Scale:** XL / High

## Summary

Adds speaker awareness to the CaseHub speech platform: offline diarization (segment a recording by speaker) and real-time speaker identification (recognise which family member is talking to the avatar). Three composable SPI interfaces in `speech-api`, dual inference paths (sherpa-onnx C API for offline, ORT for real-time), campplus as the shared embedding model, hybrid enrollment (auto-detect + explicit), and pluggable voiceprint persistence.

## Architecture

### Dual Inference Paths

Two modes share the campplus embedding model but use different inference paths:

**Offline diarization** — sherpa-onnx's `SherpaOnnxCreateOfflineSpeakerDiarization` C API bundles segmentation (pyannote), embedding extraction (campplus), and spectral clustering internally. We bind the C API via FFM/Panama, pass config + audio, get `(start, end, speaker)` segments back.

**Real-time speaker ID** — extract campplus embedding via `OnnxRuntimeLibrary` (proven path in `CosyVoice3VoiceEncoder.extractSpeakerEmbedding`), compare against registered voiceprints via cosine similarity in the `SpeakerRegistry`.

Both produce campplus 192-dim embeddings. The offline path discovers speakers via internal clustering; the real-time path identifies known speakers via registry matching. They don't need to interoperate.

### Component Diagram

```
speech-api (SPI)                    speech-sherpa (implementations)
┌─────────────────────────┐         ┌──────────────────────────────────────┐
│ SpeakerEmbeddingExtractor│────────▸│ CampplusSpeakerEmbeddingExtractor  │
│                          │         │   (ORT + campplus.onnx)              │
├─────────────────────────┤         ├──────────────────────────────────────┤
│ SpeakerRegistry          │────────▸│ CosineDistanceSpeakerRegistry        │
│   └─ VoiceprintStore     │────────▸│   └─ FileVoiceprintStore             │
├─────────────────────────┤         ├──────────────────────────────────────┤
│ SpeakerDiarizationService│────────▸│ SherpaOnnxDiarizationService         │
│                          │         │   (sherpa-onnx C API via FFM)        │
└─────────────────────────┘         └──────────────────────────────────────┘
```

### Avatar Pipeline Integration

Current flow:
```
audio → denoiser → VAD → STT → cleanup → LLM → TTS → avatar
```

New flow:
```
audio → denoiser → VAD → ┬─ STT (existing)
                          └─ SpeakerEmbeddingExtractor
                               → SpeakerRegistry.identify()
                                        ↓
                            transcript + speakerLabel
                                        ↓
                            LLM (context: "Talking to [name]")
                                        ↓
                            TTS → avatar
```

**Audio buffering for embedding extraction:** `SpeechSession` maintains its own audio ring buffer (5 seconds, ~320KB at 16kHz mono float) alongside forwarding samples to the `RecognitionStream`. `handleAudio()` writes each chunk to both the stream and the ring buffer. This is necessary because `RecognitionStream`'s internal buffer is private to the implementation and nulled on `close()` — the session cannot access it. The ring buffer is lightweight (5s is sufficient for embedding extraction; campplus needs ~1.5s minimum) and implementation-agnostic.

When recording stops, both STT final decode and embedding extraction are triggered concurrently — STT on the stream, embedding extraction on the ring buffer's contents. Embedding extraction (~50ms) completes well before STT final decode (200-500ms+), adding zero latency to the pipeline. If speaker ID fails or times out, the turn proceeds without a speaker label — graceful degradation, never a gate on the conversation.

## SPI Layer (`speech-api`)

All types are pure Java records and interfaces with zero foundation dependencies, following the established pattern of `SpeechToTextService`, `TextToSpeechService`, and `TranscriptionOptions`.

### Core Types

```java
record SpeakerEmbedding(float[] vector, int dimensions) {}

record SpeakerMatch(String name, double confidence) {}

record DiarizedSegment(long startMs, long endMs, String speakerLabel,
                       float[] samples, int sampleRate) {}

record DiarizationOptions(int numSpeakersHint,
                          float clusterThreshold) {}
```

- `DiarizedSegment` includes extracted `float[] samples` so consumers can compose with STT without re-reading and slicing the original file. The diarization C API returns `(start, end, speaker)` tuples only — `SherpaOnnxDiarizationService.diarize()` extracts samples by:
  1. Read the full audio file into `float[]` via `WavReader` (16-bit PCM WAV only) → `WavData(samples, sampleRate, channels)`
  2. Query expected sample rate from `SherpaOnnxOfflineSpeakerDiarizationGetSampleRate(handle)` (typically 16kHz)
  3. If the WAV sample rate differs from the expected rate, resample via `AudioResampler.resample(samples, wavSampleRate, expectedRate)` — same utility used in §Real-Time Embedding Extraction
  4. Pass (resampled) audio to C API, receive segment tuples
  5. For each segment: convert start/end seconds to sample indices via `(int)(start * expectedRate)`, slice the (resampled) `float[]` via `Arrays.copyOfRange()`
  6. Overlapping segments (possible with pyannote): slices may overlap — each `DiarizedSegment` gets its own independent copy of the overlapping region
  7. `DiarizedSegment.sampleRate` is set to the expected rate (post-resampling), not the original WAV rate
- `DiarizationOptions.numSpeakersHint` — set to `-1` for automatic speaker count detection via threshold-based clustering. When set to a positive integer, forces that exact number of clusters.
- `DiarizationOptions.clusterThreshold` — clustering distance threshold used when `numSpeakersHint` is `-1`. Larger values → fewer speakers. `0.0` for sherpa-onnx default.

### Interfaces

```java
interface SpeakerEmbeddingExtractor {
    SpeakerEmbedding extract(float[] samples, int sampleRate);
}

interface SpeakerRegistry {
    /** Register or re-enroll — if a speaker with the given name exists, their embedding is replaced. */
    void register(String name, SpeakerEmbedding embedding);
    Optional<SpeakerMatch> identify(SpeakerEmbedding embedding,
                                     double confidenceThreshold);
    List<String> registeredSpeakers();
    void remove(String name);
}

interface VoiceprintStore {
    void save(String name, SpeakerEmbedding embedding);
    Map<String, SpeakerEmbedding> loadAll();
    void delete(String name);
}

interface SpeakerDiarizationService {
    List<DiarizedSegment> diarize(Path audioFile,
                                  DiarizationOptions options);
}
```

### Consumer Composition

**Offline diarization + transcription:**
```java
List<DiarizedSegment> segments = diarizer.diarize(recording, options);
for (DiarizedSegment seg : segments) {
    RecognitionStream stream = stt.startStream(transcriptionOpts);
    stream.acceptSamples(seg.samples(), seg.sampleRate());
    TranscriptionResult result = stream.finalResult();
    // result.text() + seg.speakerLabel() + seg.startMs()
}
```

**Real-time speaker ID (avatar):**
```java
SpeakerEmbedding emb = extractor.extract(turnAudio, 16000);
Optional<SpeakerMatch> match = registry.identify(emb, 0.7); // threshold is a tunable default
String speaker = match.map(SpeakerMatch::name).orElse("Unknown");
```

## FFM Bindings (`speech-sherpa`)

### SherpaLibrary Additions

New `MethodHandle` fields for offline diarization C API functions:

| MethodHandle | C Function | Signature |
|-------------|------------|-----------|
| `createDiarization` | `SherpaOnnxCreateOfflineSpeakerDiarization` | `(config*) → handle*` |
| `destroyDiarization` | `SherpaOnnxDestroyOfflineSpeakerDiarization` | `(handle*) → void` |
| `diarizationGetSampleRate` | `SherpaOnnxOfflineSpeakerDiarizationGetSampleRate` | `(handle*) → int32` |
| `diarizationSetConfig` | `SherpaOnnxOfflineSpeakerDiarizationSetConfig` | `(handle*, config*) → void` |
| `diarizationProcess` | `SherpaOnnxOfflineSpeakerDiarizationProcess` | `(handle*, float*, int32) → result*` |
| `diarizationProcessWithCallback` | `SherpaOnnxOfflineSpeakerDiarizationProcessWithCallback` | `(handle*, float*, int32, callback, void*) → result*` |
| `diarizationResultGetNumSegments` | `SherpaOnnxOfflineSpeakerDiarizationResultGetNumSegments` | `(result*) → int32` |
| `diarizationResultSortByStartTime` | `SherpaOnnxOfflineSpeakerDiarizationResultSortByStartTime` | `(result*) → segments*` |
| `diarizationDestroyResult` | `SherpaOnnxOfflineSpeakerDiarizationDestroyResult` | `(result*) → void` |

### SherpaLayouts Additions

Byte offsets for `SherpaOnnxOfflineSpeakerDiarizationConfig` using the 4096-byte zero-filled allocation pattern (per GE-20260826-190329):

```java
// SherpaOnnxOfflineSpeakerDiarizationConfig — 64 bytes total (64-bit platforms)
// Segmentation sub-config (SherpaOnnxOfflineSpeakerSegmentationModelConfig, 24 bytes)
//   Pyannote sub-config (SherpaOnnxOfflineSpeakerSegmentationPyannoteModelConfig, 8 bytes)
static final long DIARIZATION_SEGMENTATION_PYANNOTE     =  0; // const char* — model path
static final long DIARIZATION_SEGMENTATION_NUM_THREADS  =  8; // int32
static final long DIARIZATION_SEGMENTATION_DEBUG        = 12; // int32
static final long DIARIZATION_SEGMENTATION_PROVIDER     = 16; // const char*

// Embedding sub-config (SherpaOnnxOfflineSpeakerEmbeddingExtractorConfig, 24 bytes)
static final long DIARIZATION_EMBEDDING_MODEL           = 24; // const char* — campplus path
static final long DIARIZATION_EMBEDDING_NUM_THREADS     = 32; // int32
static final long DIARIZATION_EMBEDDING_DEBUG           = 36; // int32
static final long DIARIZATION_EMBEDDING_PROVIDER        = 40; // const char*

// Clustering sub-config (SherpaOnnxFastClusteringConfig, 8 bytes)
static final long DIARIZATION_CLUSTERING_NUM_CLUSTERS   = 48; // int32
static final long DIARIZATION_CLUSTERING_THRESHOLD      = 52; // float

// Top-level fields
static final long DIARIZATION_MIN_DURATION_ON           = 56; // float
static final long DIARIZATION_MIN_DURATION_OFF          = 60; // float
```

Config struct is 64 bytes — significantly simpler than the STT config (~800+ bytes with 17 nested model sub-configs). Layout computed from C header `SherpaOnnxOfflineSpeakerDiarizationConfig`: pointers = 8 bytes, int32/float = 4 bytes, pointer fields aligned to 8-byte boundaries.

### SherpaOnnxDiarizationService Handle Lifecycle

**Per-instance handle** — the diarization handle is created once at construction and reused across `diarize()` calls. `createDiarization` loads the segmentation and embedding models, which is expensive (~100-500ms). Destroying and recreating per-call wastes this initialization.

**Per-call clustering config** — `DiarizationOptions` parameters (`numSpeakersHint`, `clusterThreshold`) map to the `SherpaOnnxFastClusteringConfig` sub-struct. Before each `diarizationProcess` call, `diarizationSetConfig` updates the clustering parameters on the existing handle without reloading models.

```java
class SherpaOnnxDiarizationService implements SpeakerDiarizationService, AutoCloseable {
    private final SherpaLibrary lib;
    private final MemorySegment handle;  // created once
    private final int expectedSampleRate;

    SherpaOnnxDiarizationService(SherpaLibrary lib, Path segmentationModel, Path embeddingModel) {
        this.lib = lib;
        // build config with model paths, create handle
        this.handle = createHandle(lib, segmentationModel, embeddingModel);
        this.expectedSampleRate = getSampleRate(lib, handle);
    }

    List<DiarizedSegment> diarize(Path audioFile, DiarizationOptions options) {
        updateClusteringConfig(handle, options);  // diarizationSetConfig
        // read audio, resample, process, extract segments
    }

    public void close() { lib.destroyDiarization(handle); }
}
```

**Thread safety:** the diarization handle is NOT thread-safe for concurrent `diarizationProcess` calls. If concurrent diarization is needed, use separate `SherpaOnnxDiarizationService` instances. For the current use case (offline batch processing), single-threaded access is expected.

### Result Struct Reading

`SherpaOnnxOfflineSpeakerDiarizationSegment` is a simple struct:
```
offset 0: float start   (seconds)
offset 4: float end     (seconds)
offset 8: int32 speaker (cluster index)
```

Stride: **12 bytes** (no padding — all members are 4-byte aligned, the largest member is 4 bytes, and 12 is a multiple of the struct's alignment requirement). Read via `MemorySegment.get(ValueLayout.JAVA_FLOAT, offset)` and `MemorySegment.get(ValueLayout.JAVA_INT, offset)`.

## Real-Time Embedding Extraction

### CampplusSpeakerEmbeddingExtractor

Implements `SpeakerEmbeddingExtractor` using `OnnxRuntimeLibrary` with `campplus.onnx`. Follows the exact preprocessing path proven in `CosyVoice3VoiceEncoder.extractSpeakerEmbedding`:

1. Resample input to 16kHz if needed (`AudioResampler`)
2. Compute mel spectrogram (`MelSpectrogram` with `CAMPPLUS_MEL` config: `new MelConfig(16000, 400, 160, 80, 20f, 7600f)` — sampleRate=16000, nFft=400, hopLength=160, nMels=80, fMin=20, fMax=7600)
3. Log mel transformation + mean normalisation
4. Run `campplus.onnx` via `OnnxRuntimeLibrary.Session.runFloat()`
5. Return `SpeakerEmbedding(vector, 192)`

**Thread safety:** Each extraction creates a confined `Arena` for tensor allocation and releases ORT values explicitly (per GE-20260829-c497e0 — ORT tensor handles leak despite Arena cleanup).

**Model sharing:** campplus.onnx is already loaded by `Provisioner` for CosyVoice3. The `OnnxRuntimeLibrary.Session` is thread-safe for concurrent `runFloat()` calls — create one session at startup, share across extractions.

### CosineDistanceSpeakerRegistry

Pure Java implementation of `SpeakerRegistry`:

- `ConcurrentHashMap<String, SpeakerEmbedding>` for in-memory cache (same pattern as `VoiceRegistry`)
- `VoiceprintStore` delegate for persistence — loaded at construction time via `store.loadAll()`
- `identify()`: compute cosine similarity against all registered embeddings, return best match above `confidenceThreshold`
- `register()`: add to in-memory cache + persist via `store.save()`
- `remove()`: remove from cache + persist via `store.delete()`

Cosine similarity: `dot(a, b) / (norm(a) * norm(b))`. Simple loop — 192-dim vectors, no need for SIMD.

## Voiceprint Persistence

### FileVoiceprintStore

Local file storage under `~/.casehub/voiceprints/`:

```
~/.casehub/voiceprints/
  mark.json        # { "name": "Mark", "vector": [...], "dimensions": 192 }
  sarah.json
```

- JSON serialization of `SpeakerEmbedding` + name metadata
- Atomic write: write to `{name}.json.tmp`, rename to `{name}.json`
- `loadAll()`: read all `.json` files in directory at startup
- **Concurrency model:** `ConcurrentHashMap` in-memory + atomic-rename to disk. Last-write-wins on concurrent `register()` calls for the same name. No file-level locking — adequate for the family demo use case (single device, low contention, 2-8 speakers). Production multi-device deployment would require a transactional store (the `RestVoiceprintStore` path)

### RestVoiceprintStore

SPI defined but not implemented in this issue. REST implementation is a follow-up when the platform endpoint exists:

```
GET    /api/speakers              → List<VoiceprintEntry>
PUT    /api/speakers/{name}       → register/update
DELETE /api/speakers/{name}       → remove
```

### Privacy Considerations

Speaker embeddings are biometric data under GDPR Article 9 and BIPA:
- File store uses filesystem permissions only (adequate for local demo)
- REST implementation must address encryption at rest and consent
- `SpeakerRegistry.remove()` enables right-to-erasure compliance
- Embeddings are one-way — the original audio cannot be reconstructed from a campplus vector

## Avatar Pipeline Changes (`speech-ws`)

### Structural Changes

1. **`ConversationTurn`** — add nullable speaker field:
   ```java
   record ConversationTurn(String role, String text, @Nullable String speaker) {
       public ConversationTurn {
           if (role == null || role.isBlank()) throw new IllegalArgumentException("role required");
           if (text == null) throw new IllegalArgumentException("text required");
       }
   }
   ```
   `speaker` is `@Nullable` — it is null for assistant turns, and for user turns where speaker ID was not available (too-short audio, extraction failure, no registry configured). All existing construction sites use `new ConversationTurn("user", text, null)` or `new ConversationTurn("assistant", text, null)` — the migration is mechanical. When speaker ID is available: `new ConversationTurn("user", text, speakerLabel)`.

2. **`DefaultPromptAssembler`** — format speaker context into the user prompt:
   ```java
   for (ConversationTurn turn : history) {
       String label = turn.role().equals("user") ? "User" : "Assistant";
       if (turn.speaker() != null) {
           label = turn.speaker() + " (User)";
       }
       sb.append(label).append(": ").append(turn.text()).append("\n");
   }
   ```
   When a speaker is known, the history reads `Mark (User): Hello` instead of `User: Hello`. The system prompt gets an additional line when any speakers are known: `"Speaking with: Mark, Sarah."` (distinct speaker names from history).

3. **`SpeechSession`** — when recording stops (same trigger as `finalResult()`):
   - Extract embedding from the ring buffer's contents concurrently with STT final decode
   - **Minimum audio duration:** skip extraction if the turn is shorter than 1.5 seconds — insufficient audio for a reliable embedding. Proceed without a speaker label.
   - Call `registry.identify(embedding, 0.7)` — the 0.7 confidence threshold is a tunable default, subject to empirical calibration during testing
   - If unknown: send `SpeakerPrompt` to client
   - If known: attach `speakerLabel` to the conversation turn
   - **Graceful degradation:** if extraction fails, times out, or the turn is too short, the conversation continues normally without a speaker label. Speaker ID is never a gate on the conversation flow.

4. **`SpeechWebSocket`** — inject `SpeakerEmbeddingExtractor` and `SpeakerRegistry` via CDI

### Auto-Enrollment Flow

**Enrollment state machine** in `SpeechSession`:

```
IDLE ──[identify() returns empty]──→ PENDING_ENROLLMENT
                                         │
           ┌─────────────────────────────┘
           │
PENDING_ENROLLMENT ──[SpeakerIdentify("Mark")]──→ IDLE (registered)
           │
           └──[identify() returns empty again]──→ PENDING_ENROLLMENT
                (replace stored embedding with latest, suppress duplicate prompt)
```

State fields:
- `@Nullable SpeakerEmbedding pendingEmbedding` — the embedding awaiting a name
- `boolean enrollmentPending` — whether a `SpeakerPrompt` has been sent

**Flow:**

1. User speaks → STT transcribes, embedding extracted in parallel
2. `registry.identify(embedding, 0.7)` returns `Optional.empty()` → unknown speaker
3. If `enrollmentPending` is false:
   - Store `pendingEmbedding = embedding`, set `enrollmentPending = true`
   - Send `SpeakerPrompt("I don't recognise your voice — what's your name?")` to client
4. If `enrollmentPending` is true (subsequent turn before user responded):
   - Replace `pendingEmbedding` with latest embedding (latest is most recent voice sample)
   - Do NOT send a second `SpeakerPrompt`
5. Client responds with `SpeakerIdentify("Mark")`:
   - If `pendingEmbedding != null`: call `registry.register("Mark", pendingEmbedding)`, send `SpeakerIdentified("Mark", 1.0)`
   - If `pendingEmbedding == null` (session timeout or flow moved on): send `Error("No pending enrollment — please speak again")`
   - Clear `enrollmentPending = false`, `pendingEmbedding = null`
6. Subsequent turns from this voice match against "Mark"

### New Protocol Messages

```java
// server → client: ask for speaker name
record SpeakerPrompt(String message) implements AvatarMessage {}

// client → server: provide name for enrollment
record SpeakerIdentify(String name) implements AvatarMessage {}

// server → client: confirm speaker identification
record SpeakerIdentified(String name, double confidence) implements AvatarMessage {}
```

**Message directionality and JSON encoding:**

| Record | Direction | JSON `type` value |
|--------|-----------|-------------------|
| `SpeakerPrompt` | server → client | `"speakerPrompt"` |
| `SpeakerIdentify` | client → server | `"speakerIdentify"` |
| `SpeakerIdentified` | server → client | `"speakerIdentified"` |

**`MessageCodec.encode()`** — add cases to the exhaustive switch (all three required — the sealed interface demands a case per subtype, even for client→server messages like `SpeakerIdentify`, matching the existing pattern for `Start`, `Stop`, `Text`):
```java
case AvatarMessage.SpeakerPrompt sp -> {
    obj.addProperty("type", "speakerPrompt");
    obj.addProperty("message", sp.message());
}
case AvatarMessage.SpeakerIdentify si -> {
    obj.addProperty("type", "speakerIdentify");
    obj.addProperty("name", si.name());
}
case AvatarMessage.SpeakerIdentified si -> {
    obj.addProperty("type", "speakerIdentified");
    obj.addProperty("name", si.name());
    obj.addProperty("confidence", si.confidence());
}
```

**`MessageCodec.decodeClient()`** — add case for client → server message:
```java
case "speakerIdentify" -> new AvatarMessage.SpeakerIdentify(
        obj.get("name").getAsString());
```

**`SpeechWebSocket.onText()`** — add case to the message dispatch switch:
```java
case AvatarMessage.SpeakerIdentify si -> session.handleSpeakerIdentify(si.name());
```

### Explicit Enrollment

Alternative to auto-enrollment — a deliberate "register your voice" flow:

1. Client sends `SpeakerIdentify("Mark")` **during an active recording** (between `Start` and `Stop`)
2. `SpeechSession.handleSpeakerIdentify("Mark")` sets `explicitEnrollmentName = "Mark"` — the name is queued, audio continues accumulating in the ring buffer
3. When `Stop` arrives (recording ends):
   - If `explicitEnrollmentName != null` and ring buffer has ≥ 1.5s of audio: extract embedding, call `registry.register(explicitEnrollmentName, embedding)`, send `SpeakerIdentified("Mark", 1.0)`
   - If ring buffer has < 1.5s of audio: send `Error("Recording too short for voice enrollment — need at least 1.5 seconds")`
   - Clear `explicitEnrollmentName = null`
4. If `SpeakerIdentify` arrives with **no active recording**: ignore it (send `Error("Start recording first")`)

**Distinction from auto-enrollment:** Explicit provides the name before speaking; auto-enrollment speaks first and is prompted for a name. Both produce the same `registry.register()` call. Both enrollment paths feed the same registry.

## Model Provisioning

| Model | Purpose | Source | Size | Status |
|-------|---------|--------|------|--------|
| `campplus.onnx` | Speaker embedding (192-dim) | Already provisioned for CosyVoice3 | ~7MB | Exists |
| `sherpa-onnx-pyannote-segmentation-3-0/model.onnx` | Speaker segmentation | sherpa-onnx releases (`speaker-segmentation-models`) | ~5MB | New |

**Provisioner additions:**

`ensureCampplusModel()` — ensures `campplus.onnx` is available independently of CosyVoice3:
- Target directory: `~/.casehub/models/campplus/`
- Source: HuggingFace `ayousanz/cosy-voice3-onnx` (single file: `campplus.onnx`, ~7MB)
- Expected file: `campplus.onnx`
- Uses `provisionFromHuggingFace()` (existing pattern)
- If CosyVoice3 is also provisioned, both copies coexist (negligible cost at 7MB)

`ensureDiarizationModels()` — downloads the pyannote segmentation model:
- Target directory: `~/.casehub/models/sherpa-onnx-pyannote-segmentation-3-0/`
- Source: sherpa-onnx releases → `speaker-segmentation-models/sherpa-onnx-pyannote-segmentation-3-0.tar.bz2`
- Expected file: `model.onnx`
- SHA-256 checksum verification (existing pattern)
- File-lock concurrent download protection (existing pattern)

Both methods follow the existing `ensureModel()`/`ensureTtsModel()` patterns: check-if-exists → lock → download → verify → extract.

No new native libraries. `libsherpa-onnx-c-api.dylib` (already loaded by `SherpaLibrary`) includes the diarization functions.

### Model Choice: campplus over ECAPA-TDNN

Issue #211 originally specified ECAPA-TDNN. The spec uses campplus instead, for three reasons:

1. **Zero additional model cost.** campplus.onnx (~7MB) is already provisioned for CosyVoice3 voice cloning. ECAPA-TDNN would add ~20MB of model download.
2. **Proven inference path.** `CosyVoice3VoiceEncoder.extractSpeakerEmbedding()` already runs campplus through `OnnxRuntimeLibrary.Session.runFloat()` with validated mel preprocessing. The same path is reused directly.
3. **Same embedding dimensionality.** campplus produces 192-dim embeddings, identical to ECAPA-TDNN, so the downstream registry and cosine similarity code is model-agnostic.

campplus is adequate for the family interaction use case (2-8 speakers, familiar voices, modest discrimination requirements). If testing reveals insufficient discrimination (e.g., family members with similar vocal characteristics being confused), ECAPA-TDNN is the upgrade path — the `SpeakerEmbeddingExtractor` SPI makes model swapping a single implementation change. Issue #211 will be updated to reflect campplus as the initial model.

## CDI Wiring (`speech-demo`)

New producer methods in `SpeechProducers`. All speaker-related producers are wrapped in try-catch following the existing pattern for optional services (Audio8, CosyVoice3):

```java
@Produces @ApplicationScoped
SpeakerEmbeddingExtractor embeddingExtractor() {
    // Always loads its own ORT session — no coupling to CosyVoice3
    Path campplusPath = Provisioner.ensureCampplusModel();
    OnnxRuntimeLibrary.Session session = OnnxRuntimeLibrary.load()
        .createSession(campplusPath.resolve("campplus.onnx"));
    return new CampplusSpeakerEmbeddingExtractor(session);
}

@Produces @Singleton
SpeakerRegistry speakerRegistry(VoiceprintStore store) {
    return new CosineDistanceSpeakerRegistry(store);
}

@Produces @Singleton
VoiceprintStore voiceprintStore() {
    return new FileVoiceprintStore(
        Path.of(System.getProperty("user.home"), ".casehub", "voiceprints"));
}

@Produces @ApplicationScoped
SpeakerDiarizationService diarizer() {
    Path segModelDir = Provisioner.ensureDiarizationModels();
    Path campplusDir = Provisioner.ensureCampplusModel();
    return new SherpaOnnxDiarizationService(SherpaLibrary.load(),
        segModelDir.resolve("model.onnx"),
        campplusDir.resolve("campplus.onnx"));
}
```

**Graceful degradation in `SpeechWebSocket`:** inject via `Instance<>` and check `isResolvable()` — same pattern as `agentProvider` and `ttsRegistry`:

```java
@Inject Instance<SpeakerEmbeddingExtractor> embeddingExtractor;
@Inject Instance<SpeakerRegistry> speakerRegistry;
```

If `embeddingExtractor.isResolvable()` is false (model unavailable, ORT load failure), `SpeechSession` is constructed without speaker ID capabilities — all turns proceed without speaker labels. Speaker ID is never a startup gate.

**Model independence:** `embeddingExtractor` producer calls `Provisioner.ensureCampplusModel()` which downloads `campplus.onnx` independently of CosyVoice3. The campplus model is small (~7MB) and the ORT session is cheap to create. No session sharing with CosyVoice3 — this avoids coupling and lifecycle complexity.

## Testing Strategy

| Layer | Test Type | What It Covers |
|-------|-----------|----------------|
| SPI records | Unit | `SpeakerEmbedding`, `DiarizedSegment`, `DiarizationOptions` construction |
| `CosineDistanceSpeakerRegistry` | Unit | Register, identify (match/no-match/threshold), remove, thread safety, cosine similarity edge cases (zero vector, identical vectors) |
| `FileVoiceprintStore` | Unit | Save/load/delete round-trip, atomic write, corrupt file handling |
| `CampplusSpeakerEmbeddingExtractor` | Integration | Extract embedding from known WAV → verify 192-dim output, same speaker → high similarity, different speakers → low similarity |
| `SherpaOnnxDiarizationService` | Integration | Diarize `0-four-speakers-zh.wav` (sherpa-onnx test file) → verify segment count, speaker labels, DiarizedSegment.samples non-empty |
| Diarization + STT composition | Integration | Diarize → feed segments to STT → verify speaker-attributed transcript |
| Avatar pipeline | Integration | Speak → identify → transcript includes speaker label, auto-enrollment flow |

## Deferred Work (GitHub Issues)

The following items are out of scope for this spec and must be tracked as separate GitHub issues:

1. **RestVoiceprintStore implementation** — REST-backed voiceprint persistence against the platform endpoint (`GET/PUT/DELETE /api/speakers`). Blocked on platform endpoint availability.
2. **ECAPA-TDNN upgrade path** — if campplus proves insufficient for the family interaction use case, swap `SpeakerEmbeddingExtractor` implementation to ECAPA-TDNN (~20MB, SpeechBrain ONNX export).
3. **Privacy/GDPR compliance for REST voiceprint storage** — encryption at rest, consent tracking, data subject access requests. Required before any multi-device or cloud-hosted voiceprint storage.
4. **Confidence threshold calibration** — empirical testing of the 0.7 cosine similarity threshold with real family voices. May need per-deployment tuning or adaptive thresholding.
5. **SpeechSession refactoring** — extract constructor parameters into a config record or decompose into composable pipeline stages. Pre-existing concern exacerbated by speaker ID additions.

## References

- [sherpa-onnx c-api.h](https://github.com/k2-fsa/sherpa-onnx/blob/master/sherpa-onnx/c-api/c-api.h) — diarization C API function signatures and config structs
- [sherpa-onnx speaker diarization docs](https://k2-fsa.github.io/sherpa/onnx/speaker-diarization/index.html) — model downloads and usage
- [GE-20260826-51c700] — FFM struct layout requires exact match of ALL nested model sub-configs
- [GE-20260826-190329] — oversized zero-filled allocation for FFM config structs
- [GE-20260826-3608ec] — native lib JARs contain JNI libs, not C API libs
- [GE-20260901-defe71] — SIGSEGV from ORT API version mismatch after native library swap
- [GE-20260829-c497e0] — ORT tensor handles leak despite Arena cleanup
- `SherpaLibrary.java` — existing FFM binding patterns
- `SherpaLayouts.java` — existing byte offset patterns
- `CosyVoice3VoiceEncoder.extractSpeakerEmbedding` — proven ORT campplus extraction path
- `VoiceRegistry.java` — thread-safe voice storage pattern
- `SpeechSession.java`, `ConversationTurn.java`, `PromptAssembler.java` — avatar pipeline
