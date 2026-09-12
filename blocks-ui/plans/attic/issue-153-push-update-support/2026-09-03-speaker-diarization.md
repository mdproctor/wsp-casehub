# Speaker Diarization & Identification Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/blocks#191 — Speaker diarization
**Issue group:** casehubio/blocks#191, casehubio/blocks#211 (merged)

**Goal:** Add speaker awareness — offline diarization of recordings and real-time speaker identification in the avatar pipeline.

**Architecture:** Dual inference paths sharing campplus 192-dim embeddings. Offline diarization via sherpa-onnx C API (FFM/Panama). Real-time speaker ID via ORT with campplus.onnx. Three SPI interfaces in speech-api, implementations in speech-sherpa, avatar pipeline integration in speech-ws.

**Tech Stack:** Java 22+ FFM/Panama, sherpa-onnx 1.13.6 C API, ONNX Runtime via OnnxRuntimeLibrary, campplus.onnx, pyannote segmentation model, Quarkus CDI

## Global Constraints

- JDK 22+ required (FFM/Panama)
- sherpa-onnx version pinned to 1.13.6
- SPI interfaces in speech-api must have zero foundation dependencies
- 4096-byte zero-filled config allocation pattern for FFM struct bindings (GE-20260826-190329)
- ORT tensor handles must be explicitly released (GE-20260829-c497e0)
- All code in the blocks repo: `speech-api/`, `speech-sherpa/`, `speech-ws/`, `speech-demo/`
- Use `ide-tooling` for all code navigation and structural editing
- Commit after every task with `Refs casehubio/blocks#191`

---

## Batch 1: Embedding + Registry (real-time speaker ID path)

After this batch: speaker embeddings can be extracted from audio, voiceprints can be registered and matched, and voiceprints persist to disk.

### Task 1: SPI Types + Campplus Embedding Extractor

**Files:**
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/SpeakerEmbedding.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/SpeakerMatch.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/DiarizedSegment.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/DiarizationOptions.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/SpeakerEmbeddingExtractor.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/SpeakerRegistry.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/VoiceprintStore.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/SpeakerDiarizationService.java`
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/CampplusSpeakerEmbeddingExtractor.java`
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java` — add `ensureCampplusModel()`
- Test: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/CampplusSpeakerEmbeddingExtractorTest.java`

**Interfaces:**
- Consumes: `OnnxRuntimeLibrary.Session` (existing), `MelSpectrogram` (existing), `AudioResampler` (existing), `Provisioner` (existing)
- Produces: `SpeakerEmbedding`, `SpeakerEmbeddingExtractor`, `SpeakerMatch`, `DiarizedSegment`, `DiarizationOptions`, `SpeakerRegistry`, `VoiceprintStore`, `SpeakerDiarizationService`, `CampplusSpeakerEmbeddingExtractor`

- [ ] **Step 1: Create SPI types in speech-api**

All in package `io.casehub.blocks.speech`:

```java
// SpeakerEmbedding.java
package io.casehub.blocks.speech;
public record SpeakerEmbedding(float[] vector, int dimensions) {}

// SpeakerMatch.java
package io.casehub.blocks.speech;
public record SpeakerMatch(String name, double confidence) {}

// DiarizedSegment.java
package io.casehub.blocks.speech;
public record DiarizedSegment(long startMs, long endMs, String speakerLabel,
                               float[] samples, int sampleRate) {}

// DiarizationOptions.java
package io.casehub.blocks.speech;
public record DiarizationOptions(int numSpeakersHint, float clusterThreshold) {}
```

- [ ] **Step 2: Create SPI interfaces in speech-api**

```java
// SpeakerEmbeddingExtractor.java
package io.casehub.blocks.speech;
public interface SpeakerEmbeddingExtractor {
    SpeakerEmbedding extract(float[] samples, int sampleRate);
}

// SpeakerRegistry.java
package io.casehub.blocks.speech;
import java.util.List;
import java.util.Optional;
public interface SpeakerRegistry {
    /** Register or re-enroll — if a speaker with the given name exists, their embedding is replaced. */
    void register(String name, SpeakerEmbedding embedding);
    Optional<SpeakerMatch> identify(SpeakerEmbedding embedding, double confidenceThreshold);
    List<String> registeredSpeakers();
    void remove(String name);
}

// VoiceprintStore.java
package io.casehub.blocks.speech;
import java.util.Map;
public interface VoiceprintStore {
    void save(String name, SpeakerEmbedding embedding);
    Map<String, SpeakerEmbedding> loadAll();
    void delete(String name);
}

// SpeakerDiarizationService.java
package io.casehub.blocks.speech;
import java.nio.file.Path;
import java.util.List;
public interface SpeakerDiarizationService {
    List<DiarizedSegment> diarize(Path audioFile, DiarizationOptions options);
}
```

- [ ] **Step 3: Add Provisioner.ensureCampplusModel()**

Use `ide_find_class` to navigate to `Provisioner.java`. Add method following the existing `ensureTtsModel()` pattern:

```java
public static Path ensureCampplusModel() {
    Path modelDir = modelsDir().resolve("campplus");
    Path modelFile = modelDir.resolve("campplus.onnx");
    if (Files.exists(modelFile)) return modelDir;
    provisionFromHuggingFace("ayousanz/cosy-voice3-onnx", "campplus.onnx", modelDir);
    return modelDir;
}
```

Verify by checking the existing `provisionFromHuggingFace()` calls in the same class to match the exact API.

- [ ] **Step 4: Write the failing integration test**

```java
// CampplusSpeakerEmbeddingExtractorTest.java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.SpeakerEmbedding;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import static org.junit.jupiter.api.Assertions.*;

class CampplusSpeakerEmbeddingExtractorTest {

    static CampplusSpeakerEmbeddingExtractor extractor;

    @BeforeAll
    static void setup() {
        Path campplusDir = Provisioner.ensureCampplusModel();
        OnnxRuntimeLibrary.Session session = OnnxRuntimeLibrary.load()
            .createSession(campplusDir.resolve("campplus.onnx"));
        extractor = new CampplusSpeakerEmbeddingExtractor(session);
    }

    @Test
    void extractProduces192DimEmbedding() {
        float[] silence = new float[16000 * 3]; // 3 seconds at 16kHz
        SpeakerEmbedding emb = extractor.extract(silence, 16000);
        assertEquals(192, emb.dimensions());
        assertEquals(192, emb.vector().length);
    }

    @Test
    void sameSpeakerProducesHighSimilarity() {
        float[] audio1 = generateTone(440, 3, 16000); // 3s tone at 440Hz
        float[] audio2 = generateTone(440, 3, 16000);
        SpeakerEmbedding emb1 = extractor.extract(audio1, 16000);
        SpeakerEmbedding emb2 = extractor.extract(audio2, 16000);
        double similarity = cosineSimilarity(emb1.vector(), emb2.vector());
        assertTrue(similarity > 0.9, "Same audio should produce similar embeddings");
    }

    @Test
    void differentAudioProducesDifferentEmbeddings() {
        float[] audio1 = generateTone(440, 3, 16000);
        float[] audio2 = generateTone(880, 3, 16000);
        SpeakerEmbedding emb1 = extractor.extract(audio1, 16000);
        SpeakerEmbedding emb2 = extractor.extract(audio2, 16000);
        double similarity = cosineSimilarity(emb1.vector(), emb2.vector());
        assertTrue(similarity < 0.95, "Different audio should produce different embeddings");
    }

    private static float[] generateTone(float freq, int seconds, int sampleRate) {
        float[] samples = new float[sampleRate * seconds];
        for (int i = 0; i < samples.length; i++) {
            samples[i] = (float) Math.sin(2 * Math.PI * freq * i / sampleRate);
        }
        return samples;
    }

    private static double cosineSimilarity(float[] a, float[] b) {
        double dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        return dot / (Math.sqrt(normA) * Math.sqrt(normB));
    }
}
```

- [ ] **Step 5: Run the test to verify it fails**

Run: `mvn test -pl speech-sherpa -Dtest=CampplusSpeakerEmbeddingExtractorTest -Pjdk22+`
Expected: FAIL — `CampplusSpeakerEmbeddingExtractor` class not found

- [ ] **Step 6: Implement CampplusSpeakerEmbeddingExtractor**

Use `ide_find_class` to navigate to `CosyVoice3VoiceEncoder` and read the `extractSpeakerEmbedding` method — replicate its preprocessing path.

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.SpeakerEmbedding;
import io.casehub.blocks.speech.SpeakerEmbeddingExtractor;

public class CampplusSpeakerEmbeddingExtractor implements SpeakerEmbeddingExtractor {
    private static final MelSpectrogram.MelConfig CAMPPLUS_MEL =
        new MelSpectrogram.MelConfig(16000, 400, 160, 80, 20f, 7600f);
    private static final int TARGET_SAMPLE_RATE = 16000;

    private final OnnxRuntimeLibrary.Session session;

    public CampplusSpeakerEmbeddingExtractor(OnnxRuntimeLibrary.Session session) {
        this.session = session;
    }

    @Override
    public SpeakerEmbedding extract(float[] samples, int sampleRate) {
        float[] audio = sampleRate != TARGET_SAMPLE_RATE
            ? AudioResampler.resample(samples, sampleRate, TARGET_SAMPLE_RATE)
            : samples;
        float[][] mel = MelSpectrogram.compute(audio, CAMPPLUS_MEL);
        float[][] logMel = MelSpectrogram.logMel(mel);
        float[][] normalized = MelSpectrogram.meanNormalize(logMel);
        float[] embedding = session.runFloat(
            new String[]{"input"},
            new float[][][]{normalized},
            new String[]{"output"});
        return new SpeakerEmbedding(embedding, embedding.length);
    }
}
```

Verify the exact `MelConfig` constructor and `session.runFloat()` signature by reading `CosyVoice3VoiceEncoder` via IntelliJ — the parameter names and shapes may differ. The mel → model input shape transformation (adding batch dimension, transposing) must match what campplus.onnx expects.

- [ ] **Step 7: Run the test to verify it passes**

Run: `mvn test -pl speech-sherpa -Dtest=CampplusSpeakerEmbeddingExtractorTest -Pjdk22+`
Expected: PASS (all 3 tests)

- [ ] **Step 8: Commit**

```bash
git add speech-api/src/main/java/io/casehub/blocks/speech/Speaker*.java \
        speech-api/src/main/java/io/casehub/blocks/speech/Diari*.java \
        speech-api/src/main/java/io/casehub/blocks/speech/VoiceprintStore.java \
        speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/CampplusSpeakerEmbeddingExtractor.java \
        speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java \
        speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/CampplusSpeakerEmbeddingExtractorTest.java
git commit -m "feat(#191): SPI types + campplus speaker embedding extractor

Add SpeakerEmbedding, SpeakerMatch, DiarizedSegment, DiarizationOptions
records and SpeakerEmbeddingExtractor, SpeakerRegistry, VoiceprintStore,
SpeakerDiarizationService interfaces to speech-api.

Implement CampplusSpeakerEmbeddingExtractor using OnnxRuntimeLibrary with
campplus.onnx (192-dim embeddings). Add Provisioner.ensureCampplusModel().

Refs casehubio/blocks#191"
```

---

### Task 2: Voiceprint Registry + File Persistence

**Files:**
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/CosineDistanceSpeakerRegistry.java`
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/FileVoiceprintStore.java`
- Test: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/CosineDistanceSpeakerRegistryTest.java`
- Test: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/FileVoiceprintStoreTest.java`

**Interfaces:**
- Consumes: `SpeakerEmbedding`, `SpeakerMatch`, `SpeakerRegistry`, `VoiceprintStore` (from Task 1)
- Produces: `CosineDistanceSpeakerRegistry`, `FileVoiceprintStore`

- [ ] **Step 1: Write failing tests for CosineDistanceSpeakerRegistry**

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.*;
import static org.junit.jupiter.api.Assertions.*;

class CosineDistanceSpeakerRegistryTest {

    private CosineDistanceSpeakerRegistry registry;

    @BeforeEach
    void setup() {
        VoiceprintStore store = new InMemoryVoiceprintStore();
        registry = new CosineDistanceSpeakerRegistry(store);
    }

    @Test
    void registerAndIdentify() {
        float[] vec = unitVector(192, 0);
        registry.register("Mark", new SpeakerEmbedding(vec, 192));
        Optional<SpeakerMatch> match = registry.identify(new SpeakerEmbedding(vec, 192), 0.7);
        assertTrue(match.isPresent());
        assertEquals("Mark", match.get().name());
        assertTrue(match.get().confidence() > 0.99);
    }

    @Test
    void identifyReturnsEmptyWhenNoMatch() {
        float[] vec1 = unitVector(192, 0);
        float[] vec2 = unitVector(192, 1);
        registry.register("Mark", new SpeakerEmbedding(vec1, 192));
        Optional<SpeakerMatch> match = registry.identify(new SpeakerEmbedding(vec2, 192), 0.7);
        assertTrue(match.isEmpty());
    }

    @Test
    void identifyReturnsBestMatch() {
        float[] vec = unitVector(192, 0);
        float[] similar = new float[192];
        System.arraycopy(vec, 0, similar, 0, 192);
        similar[1] = 0.1f; // slightly different
        registry.register("Mark", new SpeakerEmbedding(vec, 192));
        registry.register("Sarah", new SpeakerEmbedding(unitVector(192, 1), 192));
        Optional<SpeakerMatch> match = registry.identify(new SpeakerEmbedding(similar, 192), 0.5);
        assertTrue(match.isPresent());
        assertEquals("Mark", match.get().name());
    }

    @Test
    void removeDeletesSpeaker() {
        registry.register("Mark", new SpeakerEmbedding(unitVector(192, 0), 192));
        registry.remove("Mark");
        assertEquals(0, registry.registeredSpeakers().size());
    }

    @Test
    void reRegisterReplacesEmbedding() {
        registry.register("Mark", new SpeakerEmbedding(unitVector(192, 0), 192));
        float[] newVec = unitVector(192, 1);
        registry.register("Mark", new SpeakerEmbedding(newVec, 192));
        assertEquals(1, registry.registeredSpeakers().size());
        Optional<SpeakerMatch> match = registry.identify(new SpeakerEmbedding(newVec, 192), 0.7);
        assertTrue(match.isPresent());
    }

    private static float[] unitVector(int dims, int hotIndex) {
        float[] v = new float[dims];
        v[hotIndex % dims] = 1.0f;
        return v;
    }

    static class InMemoryVoiceprintStore implements VoiceprintStore {
        private final Map<String, SpeakerEmbedding> data = new HashMap<>();
        public void save(String name, SpeakerEmbedding e) { data.put(name, e); }
        public Map<String, SpeakerEmbedding> loadAll() { return new HashMap<>(data); }
        public void delete(String name) { data.remove(name); }
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl speech-sherpa -Dtest=CosineDistanceSpeakerRegistryTest -Pjdk22+`
Expected: FAIL — class not found

- [ ] **Step 3: Implement CosineDistanceSpeakerRegistry**

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.*;
import java.util.*;
import java.util.concurrent.ConcurrentHashMap;

public class CosineDistanceSpeakerRegistry implements SpeakerRegistry {
    private final ConcurrentHashMap<String, SpeakerEmbedding> cache = new ConcurrentHashMap<>();
    private final VoiceprintStore store;

    public CosineDistanceSpeakerRegistry(VoiceprintStore store) {
        this.store = store;
        cache.putAll(store.loadAll());
    }

    @Override
    public void register(String name, SpeakerEmbedding embedding) {
        cache.put(name, embedding);
        store.save(name, embedding);
    }

    @Override
    public Optional<SpeakerMatch> identify(SpeakerEmbedding embedding, double confidenceThreshold) {
        String bestName = null;
        double bestSimilarity = -1;
        for (var entry : cache.entrySet()) {
            double sim = cosineSimilarity(embedding.vector(), entry.getValue().vector());
            if (sim > bestSimilarity) {
                bestSimilarity = sim;
                bestName = entry.getKey();
            }
        }
        if (bestName != null && bestSimilarity >= confidenceThreshold) {
            return Optional.of(new SpeakerMatch(bestName, bestSimilarity));
        }
        return Optional.empty();
    }

    @Override
    public List<String> registeredSpeakers() {
        return List.copyOf(cache.keySet());
    }

    @Override
    public void remove(String name) {
        cache.remove(name);
        store.delete(name);
    }

    static double cosineSimilarity(float[] a, float[] b) {
        double dot = 0, normA = 0, normB = 0;
        for (int i = 0; i < a.length; i++) {
            dot += a[i] * b[i];
            normA += a[i] * a[i];
            normB += b[i] * b[i];
        }
        double denom = Math.sqrt(normA) * Math.sqrt(normB);
        return denom == 0 ? 0 : dot / denom;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn test -pl speech-sherpa -Dtest=CosineDistanceSpeakerRegistryTest -Pjdk22+`
Expected: PASS

- [ ] **Step 5: Write failing tests for FileVoiceprintStore**

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.SpeakerEmbedding;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;
import java.nio.file.Path;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class FileVoiceprintStoreTest {

    @TempDir Path tempDir;
    private FileVoiceprintStore store;

    @BeforeEach
    void setup() {
        store = new FileVoiceprintStore(tempDir);
    }

    @Test
    void saveAndLoadRoundTrip() {
        float[] vec = {1.0f, 2.0f, 3.0f};
        store.save("mark", new SpeakerEmbedding(vec, 3));
        Map<String, SpeakerEmbedding> loaded = store.loadAll();
        assertEquals(1, loaded.size());
        assertArrayEquals(vec, loaded.get("mark").vector(), 0.001f);
        assertEquals(3, loaded.get("mark").dimensions());
    }

    @Test
    void deleteRemovesFile() {
        store.save("mark", new SpeakerEmbedding(new float[]{1.0f}, 1));
        store.delete("mark");
        assertTrue(store.loadAll().isEmpty());
    }

    @Test
    void loadAllWithMultipleSpeakers() {
        store.save("mark", new SpeakerEmbedding(new float[]{1.0f}, 1));
        store.save("sarah", new SpeakerEmbedding(new float[]{2.0f}, 1));
        Map<String, SpeakerEmbedding> loaded = store.loadAll();
        assertEquals(2, loaded.size());
    }

    @Test
    void saveOverwritesExisting() {
        store.save("mark", new SpeakerEmbedding(new float[]{1.0f}, 1));
        store.save("mark", new SpeakerEmbedding(new float[]{2.0f}, 1));
        Map<String, SpeakerEmbedding> loaded = store.loadAll();
        assertEquals(1, loaded.size());
        assertArrayEquals(new float[]{2.0f}, loaded.get("mark").vector(), 0.001f);
    }
}
```

- [ ] **Step 6: Implement FileVoiceprintStore**

```java
package io.casehub.blocks.speech.sherpa;

import com.google.gson.*;
import io.casehub.blocks.speech.*;
import java.io.IOException;
import java.nio.file.*;
import java.util.*;

public class FileVoiceprintStore implements VoiceprintStore {
    private static final Gson GSON = new GsonBuilder().create();
    private final Path directory;

    public FileVoiceprintStore(Path directory) {
        this.directory = directory;
        try { Files.createDirectories(directory); } catch (IOException e) { throw new RuntimeException(e); }
    }

    @Override
    public void save(String name, SpeakerEmbedding embedding) {
        try {
            JsonObject obj = new JsonObject();
            obj.addProperty("name", name);
            obj.addProperty("dimensions", embedding.dimensions());
            JsonArray vec = new JsonArray();
            for (float v : embedding.vector()) vec.add(v);
            obj.add("vector", vec);
            Path tmp = directory.resolve(name + ".json.tmp");
            Path target = directory.resolve(name + ".json");
            Files.writeString(tmp, GSON.toJson(obj));
            Files.move(tmp, target, StandardCopyOption.REPLACE_EXISTING, StandardCopyOption.ATOMIC_MOVE);
        } catch (IOException e) { throw new RuntimeException(e); }
    }

    @Override
    public Map<String, SpeakerEmbedding> loadAll() {
        Map<String, SpeakerEmbedding> result = new HashMap<>();
        try (var stream = Files.list(directory)) {
            stream.filter(p -> p.toString().endsWith(".json")).forEach(p -> {
                try {
                    JsonObject obj = JsonParser.parseString(Files.readString(p)).getAsJsonObject();
                    String name = obj.get("name").getAsString();
                    int dims = obj.get("dimensions").getAsInt();
                    JsonArray vec = obj.getAsJsonArray("vector");
                    float[] vector = new float[vec.size()];
                    for (int i = 0; i < vec.size(); i++) vector[i] = vec.get(i).getAsFloat();
                    result.put(name, new SpeakerEmbedding(vector, dims));
                } catch (IOException e) { /* skip corrupt files */ }
            });
        } catch (IOException e) { /* empty directory */ }
        return result;
    }

    @Override
    public void delete(String name) {
        try { Files.deleteIfExists(directory.resolve(name + ".json")); } catch (IOException e) { /* ignore */ }
    }
}
```

- [ ] **Step 7: Run all tests**

Run: `mvn test -pl speech-sherpa -Dtest=CosineDistanceSpeakerRegistryTest,FileVoiceprintStoreTest -Pjdk22+`
Expected: PASS

- [ ] **Step 8: Commit**

```bash
git add speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/CosineDistanceSpeakerRegistry.java \
        speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/FileVoiceprintStore.java \
        speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/CosineDistanceSpeakerRegistryTest.java \
        speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/FileVoiceprintStoreTest.java
git commit -m "feat(#191): voiceprint registry with file persistence

CosineDistanceSpeakerRegistry with ConcurrentHashMap cache and
pluggable VoiceprintStore. FileVoiceprintStore writes JSON to
~/.casehub/voiceprints/ with atomic rename.

Refs casehubio/blocks#191"
```

---

## Batch 2: Offline Diarization

After this batch: audio recordings can be diarized into speaker-labelled segments with extracted audio.

### Task 3: FFM Diarization Bindings + SherpaOnnxDiarizationService

**Files:**
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java` — add 9 diarization MethodHandles
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java` — add diarization config offsets
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java` — add `ensureDiarizationModels()`
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxDiarizationService.java`
- Test: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxDiarizationServiceTest.java`

**Interfaces:**
- Consumes: `SherpaLibrary` (existing), `SherpaLayouts` (existing), `DiarizedSegment`, `DiarizationOptions`, `SpeakerDiarizationService` (from Task 1), `WavReader` (existing), `AudioResampler` (existing)
- Produces: `SherpaOnnxDiarizationService`

- [ ] **Step 1: Add diarization MethodHandles to SherpaLibrary**

Use `ide_find_class` to navigate to `SherpaLibrary.java`. Add 9 new MethodHandle fields following the existing pattern (e.g., `createRecognizer`). In the constructor, resolve each via `downcall()`:

```java
// Diarization
private final MethodHandle createDiarization;
private final MethodHandle destroyDiarization;
private final MethodHandle diarizationGetSampleRate;
private final MethodHandle diarizationSetConfig;
private final MethodHandle diarizationProcess;
private final MethodHandle diarizationProcessWithCallback;
private final MethodHandle diarizationResultGetNumSegments;
private final MethodHandle diarizationResultSortByStartTime;
private final MethodHandle diarizationDestroyResult;
```

In the constructor, add downcall resolutions:

```java
createDiarization = downcall("SherpaOnnxCreateOfflineSpeakerDiarization",
    FunctionDescriptor.of(ADDRESS, ADDRESS));
destroyDiarization = downcall("SherpaOnnxDestroyOfflineSpeakerDiarization",
    FunctionDescriptor.ofVoid(ADDRESS));
diarizationGetSampleRate = downcall("SherpaOnnxOfflineSpeakerDiarizationGetSampleRate",
    FunctionDescriptor.of(JAVA_INT, ADDRESS));
diarizationSetConfig = downcall("SherpaOnnxOfflineSpeakerDiarizationSetConfig",
    FunctionDescriptor.ofVoid(ADDRESS, ADDRESS));
diarizationProcess = downcall("SherpaOnnxOfflineSpeakerDiarizationProcess",
    FunctionDescriptor.of(ADDRESS, ADDRESS, ADDRESS, JAVA_INT));
diarizationProcessWithCallback = downcall("SherpaOnnxOfflineSpeakerDiarizationProcessWithCallback",
    FunctionDescriptor.of(ADDRESS, ADDRESS, ADDRESS, JAVA_INT, ADDRESS, ADDRESS));
diarizationResultGetNumSegments = downcall("SherpaOnnxOfflineSpeakerDiarizationResultGetNumSegments",
    FunctionDescriptor.of(JAVA_INT, ADDRESS));
diarizationResultSortByStartTime = downcall("SherpaOnnxOfflineSpeakerDiarizationResultSortByStartTime",
    FunctionDescriptor.of(ADDRESS, ADDRESS));
diarizationDestroyResult = downcall("SherpaOnnxOfflineSpeakerDiarizationDestroyResult",
    FunctionDescriptor.ofVoid(ADDRESS));
```

Add public accessor methods for each, matching the existing pattern.

- [ ] **Step 2: Add diarization config offsets to SherpaLayouts**

Use `ide_find_class` to navigate to `SherpaLayouts.java`. Add the byte offsets from the spec:

```java
// SherpaOnnxOfflineSpeakerDiarizationConfig — 64 bytes total
static final long DIARIZATION_SEGMENTATION_PYANNOTE    =  0;
static final long DIARIZATION_SEGMENTATION_NUM_THREADS =  8;
static final long DIARIZATION_SEGMENTATION_DEBUG       = 12;
static final long DIARIZATION_SEGMENTATION_PROVIDER    = 16;
static final long DIARIZATION_EMBEDDING_MODEL          = 24;
static final long DIARIZATION_EMBEDDING_NUM_THREADS    = 32;
static final long DIARIZATION_EMBEDDING_DEBUG          = 36;
static final long DIARIZATION_EMBEDDING_PROVIDER       = 40;
static final long DIARIZATION_CLUSTERING_NUM_CLUSTERS  = 48;
static final long DIARIZATION_CLUSTERING_THRESHOLD     = 52;
static final long DIARIZATION_MIN_DURATION_ON          = 56;
static final long DIARIZATION_MIN_DURATION_OFF         = 60;
```

**Important:** These offsets are computed from the C header. Verify by checking the struct definitions in sherpa-onnx's `c-api.h` — the pyannote sub-config may have additional fields. If the struct layout differs, the offsets must be adjusted. Test will catch misalignment via SIGSEGV or wrong results.

- [ ] **Step 3: Add Provisioner.ensureDiarizationModels()**

```java
public static Path ensureDiarizationModels() {
    Path modelDir = modelsDir().resolve("sherpa-onnx-pyannote-segmentation-3-0");
    Path modelFile = modelDir.resolve("model.onnx");
    if (Files.exists(modelFile)) return modelDir;
    provisionFromGitHub("k2-fsa/sherpa-onnx", "speaker-segmentation-models",
        "sherpa-onnx-pyannote-segmentation-3-0.tar.bz2", modelDir);
    return modelDir;
}
```

Check the existing `provisionFromGitHub()` pattern in `Provisioner` to match the exact API (tag name, asset name, extraction).

- [ ] **Step 4: Write the failing integration test**

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.*;
import org.junit.jupiter.api.BeforeAll;
import org.junit.jupiter.api.Test;
import java.nio.file.Path;
import java.util.List;
import static org.junit.jupiter.api.Assertions.*;

class SherpaOnnxDiarizationServiceTest {

    static SherpaOnnxDiarizationService service;

    @BeforeAll
    static void setup() {
        Path segModelDir = Provisioner.ensureDiarizationModels();
        Path campplusDir = Provisioner.ensureCampplusModel();
        service = new SherpaOnnxDiarizationService(
            SherpaLibrary.load(),
            segModelDir.resolve("model.onnx"),
            campplusDir.resolve("campplus.onnx"));
    }

    @Test
    void diarizeMultiSpeakerAudio() {
        // Use a multi-speaker test WAV if available, else synthesise distinct tones
        // representing different "speakers"
        Path testWav = createTestWav();
        List<DiarizedSegment> segments = service.diarize(testWav,
            new DiarizationOptions(-1, 0.0f));
        assertFalse(segments.isEmpty(), "Should produce at least one segment");
        for (DiarizedSegment seg : segments) {
            assertTrue(seg.startMs() >= 0);
            assertTrue(seg.endMs() > seg.startMs());
            assertNotNull(seg.speakerLabel());
            assertTrue(seg.samples().length > 0);
            assertEquals(16000, seg.sampleRate());
        }
    }

    @Test
    void diarizeWithKnownSpeakerCount() {
        Path testWav = createTestWav();
        List<DiarizedSegment> segments = service.diarize(testWav,
            new DiarizationOptions(2, 0.0f));
        long distinctSpeakers = segments.stream()
            .map(DiarizedSegment::speakerLabel).distinct().count();
        assertEquals(2, distinctSpeakers, "Should find exactly 2 speakers when hint is 2");
    }

    private Path createTestWav() {
        // Generate a WAV with two distinct tone segments (simulating 2 speakers)
        float[] audio = new float[16000 * 6]; // 6 seconds
        for (int i = 0; i < 16000 * 3; i++)
            audio[i] = (float) Math.sin(2 * Math.PI * 200 * i / 16000) * 0.5f;
        for (int i = 16000 * 3; i < audio.length; i++)
            audio[i] = (float) Math.sin(2 * Math.PI * 800 * i / 16000) * 0.5f;
        Path wavPath = Path.of(System.getProperty("java.io.tmpdir"), "test-diarization.wav");
        WavWriter.write(wavPath, audio, 16000, 1);
        return wavPath;
    }
}
```

- [ ] **Step 5: Run test to verify it fails**

Run: `mvn test -pl speech-sherpa -Dtest=SherpaOnnxDiarizationServiceTest -Pjdk22+`
Expected: FAIL — `SherpaOnnxDiarizationService` class not found

- [ ] **Step 6: Implement SherpaOnnxDiarizationService**

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.*;
import java.lang.foreign.*;
import java.nio.file.Path;
import java.util.*;

public class SherpaOnnxDiarizationService implements SpeakerDiarizationService, AutoCloseable {
    private final SherpaLibrary lib;
    private final MemorySegment handle;
    private final int expectedSampleRate;
    private final Arena globalArena = Arena.ofShared();

    public SherpaOnnxDiarizationService(SherpaLibrary lib, Path segmentationModel, Path embeddingModel) {
        this.lib = lib;
        this.handle = createHandle(lib, segmentationModel, embeddingModel);
        this.expectedSampleRate = (int) lib.diarizationGetSampleRate().invokeExact(handle);
    }

    @Override
    public List<DiarizedSegment> diarize(Path audioFile, DiarizationOptions options) {
        WavReader.WavData wav = WavReader.read(audioFile);
        float[] audio = wav.sampleRate() != expectedSampleRate
            ? AudioResampler.resample(wav.samples(), wav.sampleRate(), expectedSampleRate)
            : wav.samples();

        updateClusteringConfig(options);

        try (Arena arena = Arena.ofConfined()) {
            MemorySegment samples = arena.allocateFrom(ValueLayout.JAVA_FLOAT, audio);
            MemorySegment result = (MemorySegment) lib.diarizationProcess()
                .invokeExact(handle, samples, audio.length);
            try {
                int numSegments = (int) lib.diarizationResultGetNumSegments().invokeExact(result);
                MemorySegment sorted = (MemorySegment) lib.diarizationResultSortByStartTime()
                    .invokeExact(result);
                sorted = sorted.reinterpret((long) numSegments * 12);

                List<DiarizedSegment> segments = new ArrayList<>(numSegments);
                for (int i = 0; i < numSegments; i++) {
                    long offset = (long) i * 12;
                    float start = sorted.get(ValueLayout.JAVA_FLOAT, offset);
                    float end = sorted.get(ValueLayout.JAVA_FLOAT, offset + 4);
                    int speaker = sorted.get(ValueLayout.JAVA_INT, offset + 8);
                    int startIdx = (int) (start * expectedSampleRate);
                    int endIdx = Math.min((int) (end * expectedSampleRate), audio.length);
                    float[] segSamples = Arrays.copyOfRange(audio, startIdx, endIdx);
                    segments.add(new DiarizedSegment(
                        (long) (start * 1000), (long) (end * 1000),
                        "speaker_" + speaker, segSamples, expectedSampleRate));
                }
                return segments;
            } finally {
                lib.diarizationDestroyResult().invokeExact(result);
            }
        } catch (Throwable e) {
            throw new RuntimeException("Diarization failed", e);
        }
    }

    private MemorySegment createHandle(SherpaLibrary lib, Path segModel, Path embModel) {
        try (Arena arena = Arena.ofConfined()) {
            MemorySegment config = arena.allocate(SherpaLayouts.CONFIG_ALLOC_SIZE);
            config.fill((byte) 0);
            config.set(ValueLayout.ADDRESS, SherpaLayouts.DIARIZATION_SEGMENTATION_PYANNOTE,
                globalArena.allocateFrom(segModel.toString()));
            config.set(ValueLayout.ADDRESS, SherpaLayouts.DIARIZATION_EMBEDDING_MODEL,
                globalArena.allocateFrom(embModel.toString()));
            config.set(ValueLayout.JAVA_INT, SherpaLayouts.DIARIZATION_SEGMENTATION_NUM_THREADS, 2);
            config.set(ValueLayout.JAVA_INT, SherpaLayouts.DIARIZATION_EMBEDDING_NUM_THREADS, 2);
            config.set(ValueLayout.JAVA_INT, SherpaLayouts.DIARIZATION_CLUSTERING_NUM_CLUSTERS, -1);
            config.set(ValueLayout.JAVA_FLOAT, SherpaLayouts.DIARIZATION_CLUSTERING_THRESHOLD, 0.5f);
            return (MemorySegment) lib.createDiarization().invokeExact(config);
        } catch (Throwable e) {
            throw new RuntimeException("Failed to create diarization handle", e);
        }
    }

    private void updateClusteringConfig(DiarizationOptions options) {
        try (Arena arena = Arena.ofConfined()) {
            MemorySegment config = arena.allocate(SherpaLayouts.CONFIG_ALLOC_SIZE);
            config.fill((byte) 0);
            config.set(ValueLayout.JAVA_INT, SherpaLayouts.DIARIZATION_CLUSTERING_NUM_CLUSTERS,
                options.numSpeakersHint());
            if (options.clusterThreshold() > 0) {
                config.set(ValueLayout.JAVA_FLOAT, SherpaLayouts.DIARIZATION_CLUSTERING_THRESHOLD,
                    options.clusterThreshold());
            }
            lib.diarizationSetConfig().invokeExact(handle, config);
        } catch (Throwable e) {
            throw new RuntimeException("Failed to update clustering config", e);
        }
    }

    @Override
    public void close() {
        try {
            lib.destroyDiarization().invokeExact(handle);
            globalArena.close();
        } catch (Throwable e) {
            throw new RuntimeException(e);
        }
    }
}
```

**Critical:** The `MethodHandle.invokeExact()` calls require exact return type casts — check the `FunctionDescriptor` signatures from Step 1. The segment stride (12 bytes) must match the C struct layout — if you get corrupt data, check padding.

- [ ] **Step 7: Run test to verify it passes**

Run: `mvn test -pl speech-sherpa -Dtest=SherpaOnnxDiarizationServiceTest -Pjdk22+`
Expected: PASS

If SIGSEGV: the byte offsets in `SherpaLayouts` are wrong. Debug by printing the config struct size and comparing with the C header. Fall back to the 4096-byte allocation pattern with manually verified offsets.

- [ ] **Step 8: Commit**

```bash
git add speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java \
        speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java \
        speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxDiarizationService.java \
        speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java \
        speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxDiarizationServiceTest.java
git commit -m "feat(#191): offline diarization via sherpa-onnx C API

Add 9 diarization MethodHandles to SherpaLibrary, config byte offsets
to SherpaLayouts, SherpaOnnxDiarizationService with per-instance handle
and per-call clustering config. Provisioner.ensureDiarizationModels()
downloads pyannote segmentation model.

Refs casehubio/blocks#191"
```

---

## Batch 3: Avatar Integration

After this batch: the avatar recognises family members during conversations, with auto-enrollment and explicit enrollment.

### Task 4: Protocol Messages + ConversationTurn + PromptAssembler

**Files:**
- Modify: `speech-ws/src/main/java/io/casehub/blocks/speech/ws/protocol/AvatarMessage.java` — add 3 new record permits
- Modify: `speech-ws/src/main/java/io/casehub/blocks/speech/ws/protocol/MessageCodec.java` — add encode/decode cases
- Modify: `speech-ws/src/main/java/io/casehub/blocks/speech/ws/protocol/ConversationTurn.java` — add nullable speaker field
- Modify: `speech-ws/src/main/java/io/casehub/blocks/speech/ws/DefaultPromptAssembler.java` (or wherever PromptAssembler is implemented) — speaker-aware formatting
- Test: `speech-ws/src/test/java/io/casehub/blocks/speech/ws/protocol/MessageCodecTest.java` — add speaker message tests
- Test: `speech-ws/src/test/java/io/casehub/blocks/speech/ws/protocol/ConversationTurnTest.java`

**Interfaces:**
- Consumes: `AvatarMessage` sealed interface (existing), `MessageCodec` (existing), `ConversationTurn` (existing)
- Produces: `AvatarMessage.SpeakerPrompt`, `AvatarMessage.SpeakerIdentify`, `AvatarMessage.SpeakerIdentified`, updated `ConversationTurn(role, text, speaker)`

- [ ] **Step 1: Write failing tests for new protocol messages**

Use `ide_find_class` to navigate to `MessageCodec` and read the existing encode/decode test patterns.

```java
@Test
void encodeSpeakerPrompt() {
    String json = MessageCodec.encode(new AvatarMessage.SpeakerPrompt("What's your name?"));
    JsonObject obj = JsonParser.parseString(json).getAsJsonObject();
    assertEquals("speakerPrompt", obj.get("type").getAsString());
    assertEquals("What's your name?", obj.get("message").getAsString());
}

@Test
void decodeSpeakerIdentify() {
    AvatarMessage msg = MessageCodec.decodeClient("{\"type\":\"speakerIdentify\",\"name\":\"Mark\"}");
    assertInstanceOf(AvatarMessage.SpeakerIdentify.class, msg);
    assertEquals("Mark", ((AvatarMessage.SpeakerIdentify) msg).name());
}

@Test
void encodeSpeakerIdentified() {
    String json = MessageCodec.encode(new AvatarMessage.SpeakerIdentified("Mark", 0.95));
    JsonObject obj = JsonParser.parseString(json).getAsJsonObject();
    assertEquals("speakerIdentified", obj.get("type").getAsString());
    assertEquals("Mark", obj.get("name").getAsString());
    assertEquals(0.95, obj.get("confidence").getAsDouble(), 0.01);
}
```

- [ ] **Step 2: Write failing test for ConversationTurn with speaker**

```java
@Test
void conversationTurnWithSpeaker() {
    var turn = new ConversationTurn("user", "hello", "Mark");
    assertEquals("Mark", turn.speaker());
}

@Test
void conversationTurnWithoutSpeaker() {
    var turn = new ConversationTurn("user", "hello", null);
    assertNull(turn.speaker());
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `mvn test -pl speech-ws -Dtest=MessageCodecTest,ConversationTurnTest`
Expected: FAIL

- [ ] **Step 4: Add protocol messages to AvatarMessage**

Use `ide_find_class` to navigate to `AvatarMessage.java`. Add three new records to the sealed interface's permits:

```java
record SpeakerPrompt(String message) implements AvatarMessage {}
record SpeakerIdentify(String name) implements AvatarMessage {}
record SpeakerIdentified(String name, double confidence) implements AvatarMessage {}
```

- [ ] **Step 5: Add encode/decode cases to MessageCodec**

Add the three encode cases from the spec (lines 387-399) to the `encode()` switch. Add the decode case (lines 404-405) to `decodeClient()`. Use `ide_find_references` on `encode()` to verify all call sites.

- [ ] **Step 6: Update ConversationTurn**

Add `@Nullable String speaker` as the third field. Update the compact constructor to accept null speaker. Use `ide_find_references` on `ConversationTurn` to find all construction sites and add the third `null` argument.

- [ ] **Step 7: Update PromptAssembler**

Use `ide_find_class` to navigate to the PromptAssembler implementation. Update the history formatting to include speaker labels when present, per the spec (lines 309-317).

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn test -pl speech-ws -Dtest=MessageCodecTest,ConversationTurnTest`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add speech-ws/
git commit -m "feat(#191): speaker protocol messages + ConversationTurn speaker field

Add SpeakerPrompt, SpeakerIdentify, SpeakerIdentified to AvatarMessage.
Add nullable speaker field to ConversationTurn. Update MessageCodec
encode/decode. Update PromptAssembler for speaker-aware formatting.

Refs casehubio/blocks#191"
```

---

### Task 5: SpeechSession Speaker ID + Enrollment + CDI Wiring

**Files:**
- Modify: `speech-ws/src/main/java/io/casehub/blocks/speech/ws/SpeechSession.java` — ring buffer, speaker ID, enrollment state machine
- Modify: `speech-ws/src/main/java/io/casehub/blocks/speech/ws/SpeechWebSocket.java` — inject speaker services, dispatch SpeakerIdentify
- Modify: `speech-demo/` (or `examples/avatar-demo/`) `SpeechProducers.java` — add CDI producers for embedding extractor, registry, store, diarizer
- Test: `speech-ws/src/test/java/io/casehub/blocks/speech/ws/SpeechSessionSpeakerTest.java`

**Interfaces:**
- Consumes: `SpeakerEmbeddingExtractor`, `SpeakerRegistry`, `VoiceprintStore`, `SpeakerDiarizationService` (from Tasks 1-3), `AvatarMessage.SpeakerPrompt`, `AvatarMessage.SpeakerIdentify`, `AvatarMessage.SpeakerIdentified`, `ConversationTurn` with speaker (from Task 4)
- Produces: end-to-end avatar speaker identification

- [ ] **Step 1: Write failing test for ring buffer and speaker identification**

```java
package io.casehub.blocks.speech.ws;

import io.casehub.blocks.speech.*;
import org.junit.jupiter.api.Test;
import java.util.Optional;
import static org.junit.jupiter.api.Assertions.*;

class SpeechSessionSpeakerTest {

    @Test
    void identifiesKnownSpeaker() {
        var extractor = new StubEmbeddingExtractor(new float[]{1, 0, 0});
        var registry = new StubSpeakerRegistry("Mark", new float[]{1, 0, 0});
        // Construct SpeechSession with speaker services
        // Simulate: handleAudio (accumulate), handleStop (extract + identify)
        // Assert: ConversationTurn has speaker = "Mark"
    }

    @Test
    void skipsSpeakerIdForShortAudio() {
        var extractor = new StubEmbeddingExtractor(new float[]{1, 0, 0});
        var registry = new StubSpeakerRegistry("Mark", new float[]{1, 0, 0});
        // Simulate: handleAudio with < 1.5s of audio
        // Assert: ConversationTurn has speaker = null
    }

    @Test
    void autoEnrollmentSendsPromptForUnknownSpeaker() {
        var extractor = new StubEmbeddingExtractor(new float[]{1, 0, 0});
        var registry = new StubSpeakerRegistry(); // empty
        // Simulate: handleAudio, handleStop
        // Assert: SpeakerPrompt message was sent to client
    }
}
```

The test stubs implement `SpeakerEmbeddingExtractor` and `SpeakerRegistry` with deterministic behaviour. Use the actual `SpeechSession` constructor — check its current signature via `ide_find_class` and adapt the test to match.

- [ ] **Step 2: Run tests to verify they fail**

Run: `mvn test -pl speech-ws -Dtest=SpeechSessionSpeakerTest`
Expected: FAIL

- [ ] **Step 3: Add ring buffer to SpeechSession**

Use `ide_find_class` to navigate to `SpeechSession.java`. Add:

```java
private static final int RING_BUFFER_SECONDS = 5;
private static final int RING_BUFFER_SIZE = 16000 * RING_BUFFER_SECONDS;
private static final int MIN_SAMPLES_FOR_EMBEDDING = 16000 * 3 / 2; // 1.5 seconds
private final float[] ringBuffer = new float[RING_BUFFER_SIZE];
private int ringBufferPos = 0;

private final @Nullable SpeakerEmbeddingExtractor embeddingExtractor;
private final @Nullable SpeakerRegistry speakerRegistry;
private @Nullable SpeakerEmbedding pendingEmbedding;
private boolean enrollmentPending;
private @Nullable String explicitEnrollmentName;
```

In `handleAudio()`, after forwarding to `stream.acceptSamples()`, copy to ring buffer:

```java
int toCopy = Math.min(samples.length, RING_BUFFER_SIZE - ringBufferPos);
System.arraycopy(samples, 0, ringBuffer, ringBufferPos, toCopy);
ringBufferPos += toCopy;
if (ringBufferPos >= RING_BUFFER_SIZE) ringBufferPos = RING_BUFFER_SIZE; // cap, don't wrap
```

- [ ] **Step 4: Add speaker identification to the stop handler**

In the method that handles recording stop (where `finalResult()` is called), add concurrent speaker ID:

```java
// After recording stops, before LLM call:
String speakerLabel = null;
if (embeddingExtractor != null && speakerRegistry != null && ringBufferPos >= MIN_SAMPLES_FOR_EMBEDDING) {
    float[] audioForEmbedding = Arrays.copyOf(ringBuffer, ringBufferPos);
    SpeakerEmbedding embedding = embeddingExtractor.extract(audioForEmbedding, 16000);

    if (explicitEnrollmentName != null) {
        speakerRegistry.register(explicitEnrollmentName, embedding);
        send(new AvatarMessage.SpeakerIdentified(explicitEnrollmentName, 1.0));
        speakerLabel = explicitEnrollmentName;
        explicitEnrollmentName = null;
    } else {
        Optional<SpeakerMatch> match = speakerRegistry.identify(embedding, 0.7);
        if (match.isPresent()) {
            speakerLabel = match.get().name();
            send(new AvatarMessage.SpeakerIdentified(speakerLabel, match.get().confidence()));
        } else {
            pendingEmbedding = embedding;
            if (!enrollmentPending) {
                enrollmentPending = true;
                send(new AvatarMessage.SpeakerPrompt("I don't recognise your voice — what's your name?"));
            }
        }
    }
}
ringBufferPos = 0; // reset for next turn
```

Pass `speakerLabel` to `ConversationTurn` construction.

- [ ] **Step 5: Add handleSpeakerIdentify method**

```java
public void handleSpeakerIdentify(String name) {
    if (pendingEmbedding != null && speakerRegistry != null) {
        speakerRegistry.register(name, pendingEmbedding);
        send(new AvatarMessage.SpeakerIdentified(name, 1.0));
        pendingEmbedding = null;
        enrollmentPending = false;
    } else if (explicitEnrollmentName == null) {
        explicitEnrollmentName = name;
    }
}
```

- [ ] **Step 6: Update SpeechWebSocket to inject speaker services and dispatch**

Use `ide_find_class` to navigate to `SpeechWebSocket.java`. Add:

```java
@Inject Instance<SpeakerEmbeddingExtractor> embeddingExtractor;
@Inject Instance<SpeakerRegistry> speakerRegistry;
```

In session construction, pass the resolved instances (or null if not resolvable). In the message dispatch switch, add:

```java
case AvatarMessage.SpeakerIdentify si -> session.handleSpeakerIdentify(si.name());
```

- [ ] **Step 7: Add CDI producers to SpeechProducers**

Use `ide_find_class` to navigate to `SpeechProducers.java`. Add the four producer methods from the spec (lines 469-496).

- [ ] **Step 8: Run tests to verify they pass**

Run: `mvn test -pl speech-ws -Dtest=SpeechSessionSpeakerTest`
Expected: PASS

Also run the full test suite to check for regressions:
Run: `mvn test -pl speech-api,speech-sherpa,speech-ws -Pjdk22+`
Expected: PASS

- [ ] **Step 9: Commit**

```bash
git add speech-ws/ speech-demo/
git commit -m "feat(#191): avatar speaker identification + enrollment

SpeechSession ring buffer for audio accumulation, concurrent speaker
embedding extraction, auto-enrollment state machine, explicit enrollment
flow. CDI producers for embedding extractor, registry, file store, and
diarization service. Graceful degradation when speaker services unavailable.

Refs casehubio/blocks#191"
```

---

## References

- [2026-09-03-speaker-diarization-design.md] — design spec this plan implements
- [SherpaLibrary.java] — existing FFM binding patterns (MethodHandle fields, downcall())
- [SherpaLayouts.java] — existing byte offset patterns (4096-byte zero-filled config)
- [CosyVoice3VoiceEncoder.java] — proven campplus ORT extraction path (extractSpeakerEmbedding)
- [VoiceRegistry.java] — thread-safe voice storage pattern (ConcurrentHashMap)
- [SpeechSession.java] — avatar pipeline orchestrator (handleAudio, finalResult flow)
- [SpeechWebSocket.java] — CDI injection, message dispatch
- [MessageCodec.java] — AvatarMessage JSON encoding/decoding
- [ConversationTurn.java] — conversation history record
- [Provisioner.java] — model download/caching patterns
- [GE-20260826-190329] — oversized zero-filled allocation for FFM config structs
- [GE-20260829-c497e0] — ORT tensor handles leak despite Arena cleanup
- [GE-20260826-51c700] — FFM struct layout requires exact match of nested sub-configs
- casehubio/blocks#191 — Speaker diarization
- casehubio/blocks#211 — Speaker identification (merged)
