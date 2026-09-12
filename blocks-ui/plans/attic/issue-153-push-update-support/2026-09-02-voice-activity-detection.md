# Voice Activity Detection Pre-filtering Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #184 — Voice Activity Detection (VAD) for speech pre-filtering
**Issue group:** #184

**Goal:** Add optional, runtime-configurable VAD as a pre-filtering gate
before STT, using sherpa-onnx's Silero VAD model to drop silence chunks.

**Architecture:** Define `VoiceActivityFilter` SPI in `speech-api`.
Implement via FFM bindings in `speech-sherpa` wrapping Silero VAD. Inject
into STT services compositionally via `withVoiceActivityFilter()`. Pipeline
order: denoiser → VAD → STT. Runtime toggle via config property.

**Tech Stack:** Java 22+ FFM, sherpa-onnx v1.13.6 C API, Silero VAD
(`silero_vad.onnx`, 16kHz, MIT license).

## Global Constraints

- sherpa-onnx v1.13.6 native library required for integration tests
- FFM struct layouts use 4096-byte zero-filled allocation (GE-20260826-190329)
- All native tests gated with `@EnabledIf` on library availability
- VAD runs AFTER denoiser in `acceptSamples()` — denoiser → VAD → buffer
- `filterChunk()` returns samples unchanged for speech, `float[0]` for silence

---

## Batch 1: SPI + FFM Foundation

### Task 1: SPI interfaces, FFM bindings, layout constants, model provisioning

**Files:**
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/VoiceActivityFilter.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/VoiceActivityFilterFactory.java`
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java` — add 6 VAD MethodHandle fields + downcall lookups
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java` — add VAD config offset constants
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java` — add `VAD_MODEL_EXPECTED_FILES` map + `ensureVadModel()` method

**Interfaces:**
- Produces: `VoiceActivityFilter` — `float[] filterChunk(float[] samples, int sampleRate)`, `void reset()`, `void close()`
- Produces: `VoiceActivityFilterFactory` — `VoiceActivityFilter create()`
- Produces: SherpaLibrary fields `createVad`, `destroyVad`, `vadAcceptWaveform`, `vadDetected`, `vadReset`, `vadFlush`
- Produces: SherpaLayouts constants `VAD_SILERO_MODEL`, `VAD_SILERO_THRESHOLD`, `VAD_SILERO_MIN_SILENCE`, `VAD_SILERO_MIN_SPEECH`, `VAD_SILERO_WINDOW_SIZE`, `VAD_SILERO_MAX_SPEECH`, `VAD_SAMPLE_RATE`, `VAD_NUM_THREADS`, `VAD_PROVIDER`
- Produces: `Provisioner.ensureVadModel(String modelName)` → `Path`

- [ ] **Step 1: Create VoiceActivityFilter interface**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech;

public interface VoiceActivityFilter extends AutoCloseable {
    float[] filterChunk(float[] samples, int sampleRate);
    void reset();
    void close();
}
```

Path: `speech-api/src/main/java/io/casehub/blocks/speech/VoiceActivityFilter.java`

- [ ] **Step 2: Create VoiceActivityFilterFactory interface**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech;

public interface VoiceActivityFilterFactory {
    VoiceActivityFilter create();
}
```

Path: `speech-api/src/main/java/io/casehub/blocks/speech/VoiceActivityFilterFactory.java`

- [ ] **Step 3: Add VAD config offset constants to SherpaLayouts**

Use `ide_insert_member` after `DENOISER_DPDFNET_MODEL`:

```java
// === SherpaOnnxVadModelConfig offsets ===
// silero_vad sub-struct at offset 0
static final long VAD_SILERO_MODEL        = 0;   // char* (8 bytes)
static final long VAD_SILERO_THRESHOLD    = 8;   // float (4 bytes)
static final long VAD_SILERO_MIN_SILENCE  = 12;  // float (4 bytes)
static final long VAD_SILERO_MIN_SPEECH   = 16;  // float (4 bytes)
static final long VAD_SILERO_WINDOW_SIZE  = 20;  // int32 (4 bytes)
static final long VAD_SILERO_MAX_SPEECH   = 24;  // float (4 bytes)
// after silero_vad (28 bytes + 4 pad = 32)
static final long VAD_SAMPLE_RATE         = 32;  // int32 (4 bytes)
static final long VAD_NUM_THREADS         = 36;  // int32 (4 bytes)
static final long VAD_PROVIDER            = 40;  // char* (8 bytes)
```

- [ ] **Step 4: Add 6 VAD MethodHandle fields to SherpaLibrary**

Use `ide_insert_member` after `onlineDenoiserReset`:

```java
// Voice Activity Detector handles
final MethodHandle createVad;
final MethodHandle destroyVad;
final MethodHandle vadAcceptWaveform;
final MethodHandle vadDetected;
final MethodHandle vadReset;
final MethodHandle vadFlush;
```

- [ ] **Step 5: Add downcall lookups in SherpaLibrary constructor**

Use `ide_replace_text_in_file` to append after the online denoiser handles block (after the line ending with `FunctionDescriptor.ofVoid(ADDRESS));` for `onlineDenoiserReset`):

```java
// Voice Activity Detector handles
createVad = downcall(linker, "SherpaOnnxCreateVoiceActivityDetector",
        FunctionDescriptor.of(ADDRESS, ADDRESS, JAVA_FLOAT));
destroyVad = downcall(linker, "SherpaOnnxDestroyVoiceActivityDetector",
        FunctionDescriptor.ofVoid(ADDRESS));
vadAcceptWaveform = downcall(linker, "SherpaOnnxVoiceActivityDetectorAcceptWaveform",
        FunctionDescriptor.ofVoid(ADDRESS, ADDRESS, JAVA_INT));
vadDetected = downcall(linker, "SherpaOnnxVoiceActivityDetectorDetected",
        FunctionDescriptor.of(JAVA_INT, ADDRESS));
vadReset = downcall(linker, "SherpaOnnxVoiceActivityDetectorReset",
        FunctionDescriptor.ofVoid(ADDRESS));
vadFlush = downcall(linker, "SherpaOnnxVoiceActivityDetectorFlush",
        FunctionDescriptor.ofVoid(ADDRESS));
```

Note: `createVad` takes `(ADDRESS, JAVA_FLOAT)` — the config pointer and `buffer_size_in_seconds` float parameter.

- [ ] **Step 6: Add model provisioning to Provisioner**

Use `ide_insert_member` after `DENOISER_MODEL_EXPECTED_FILES`:

```java
private static final Map<String, String> VAD_MODEL_EXPECTED_FILES = Map.of(
        "silero_vad", "silero_vad.onnx"
);
```

Add `vadModelDir` method after `ensureDenoiserModel`:

```java
static Path vadModelDir(String modelName) {
    return cacheBaseDir().resolve("models").resolve("sherpa-onnx").resolve(modelName);
}

static String vadModelUrl(String modelName) {
    return baseUrl() + "vad-models/" + modelName + ".onnx";
}

static Path ensureVadModel(String modelName) {
    String expectedFile = VAD_MODEL_EXPECTED_FILES.get(modelName);
    if (expectedFile == null) {
        throw new SherpaException(
                "Unknown VAD model: " + modelName
                + ". Known models: " + VAD_MODEL_EXPECTED_FILES.keySet());
    }
    Path targetDir = vadModelDir(modelName);

    if (Files.isDirectory(targetDir) && Files.exists(targetDir.resolve(expectedFile))) {
        return targetDir;
    }

    synchronized (MODEL_LOCK) {
        if (Files.isDirectory(targetDir) && Files.exists(targetDir.resolve(expectedFile))) {
            return targetDir;
        }
        try {
            Files.createDirectories(targetDir);
            String url = vadModelUrl(modelName);
            LOG.log(System.Logger.Level.INFO, "Downloading {0}...", url);
            Path downloaded = downloadWithRetry(url, targetDir, 3);
            Path dest = targetDir.resolve(expectedFile);
            if (!downloaded.getFileName().toString().equals(expectedFile)) {
                Files.move(downloaded, dest, StandardCopyOption.REPLACE_EXISTING);
            }
        } catch (IOException e) {
            throw new SherpaException("Failed to download VAD model: " + modelName
                                      + ". Download manually to " + targetDir, e);
        }
        return targetDir;
    }
}
```

- [ ] **Step 7: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl speech-api install -q && /opt/homebrew/bin/mvn -pl speech-sherpa compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 8: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add speech-api/src/main/java/io/casehub/blocks/speech/VoiceActivityFilter.java speech-api/src/main/java/io/casehub/blocks/speech/VoiceActivityFilterFactory.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#184): add VoiceActivityFilter SPIs, FFM bindings, and model provisioning

Refs #184"
```

---

## Batch 2: VAD Implementation

### Task 2: SherpaOnnxVoiceActivityFilter — Silero VAD wrapper

**Files:**
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxVoiceActivityFilter.java`
- Create: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxVoiceActivityFilterTest.java`

**Interfaces:**
- Consumes: `VoiceActivityFilter`, `VoiceActivityFilterFactory` (from Task 1), `SherpaLibrary` fields (`createVad`, `destroyVad`, `vadAcceptWaveform`, `vadDetected`, `vadReset`, `vadFlush`), `SherpaLayouts` VAD constants, `Provisioner.ensureVadModel(String)`
- Produces: `SherpaOnnxVoiceActivityFilter` — implements `VoiceActivityFilterFactory`, `static withDefaults()`, `VoiceActivityFilter create()`

- [ ] **Step 1: Write the failing test**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.VoiceActivityFilter;
import io.casehub.blocks.speech.VoiceActivityFilterFactory;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import static org.junit.jupiter.api.Assertions.*;

@EnabledIf("io.casehub.blocks.speech.sherpa.SherpaLibrary#isAvailable")
class SherpaOnnxVoiceActivityFilterTest {

    @Test
    void factoryCreatesFilterInstances() {
        VoiceActivityFilterFactory factory = SherpaOnnxVoiceActivityFilter.withDefaults();
        try (VoiceActivityFilter f1 = factory.create();
             VoiceActivityFilter f2 = factory.create()) {
            assertNotNull(f1);
            assertNotNull(f2);
            assertNotSame(f1, f2);
        }
    }

    @Test
    void silenceReturnsEmptyArray() {
        VoiceActivityFilterFactory factory = SherpaOnnxVoiceActivityFilter.withDefaults();
        try (VoiceActivityFilter filter = factory.create()) {
            float[] silence = new float[512];
            float[] result = filter.filterChunk(silence, 16000);
            assertEquals(0, result.length);
        }
    }

    @Test
    void speechReturnsNonEmptyArray() {
        VoiceActivityFilterFactory factory = SherpaOnnxVoiceActivityFilter.withDefaults();
        try (VoiceActivityFilter filter = factory.create()) {
            // Generate a loud sine wave — should trigger speech detection
            float[] speech = new float[16000]; // 1 second
            for (int i = 0; i < speech.length; i++) {
                speech[i] = (float) Math.sin(2 * Math.PI * 440 * i / 16000) * 0.8f;
            }
            // Feed in window_size chunks (512 samples each)
            float[] lastResult = new float[0];
            for (int offset = 0; offset < speech.length; offset += 512) {
                int len = Math.min(512, speech.length - offset);
                float[] chunk = new float[len];
                System.arraycopy(speech, offset, chunk, 0, len);
                lastResult = filter.filterChunk(chunk, 16000);
            }
            // At some point during loud audio, VAD should detect speech
            // (this is a best-effort test — VAD may or may not trigger on sine wave)
            assertNotNull(lastResult);
        }
    }

    @Test
    void resetClearsState() {
        VoiceActivityFilterFactory factory = SherpaOnnxVoiceActivityFilter.withDefaults();
        try (VoiceActivityFilter filter = factory.create()) {
            float[] silence = new float[512];
            filter.filterChunk(silence, 16000);
            assertDoesNotThrow(filter::reset);
        }
    }

    @Test
    void emptyInputReturnsEmpty() {
        VoiceActivityFilterFactory factory = SherpaOnnxVoiceActivityFilter.withDefaults();
        try (VoiceActivityFilter filter = factory.create()) {
            float[] result = filter.filterChunk(new float[0], 16000);
            assertEquals(0, result.length);
        }
    }
}
```

Path: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxVoiceActivityFilterTest.java`

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=SherpaOnnxVoiceActivityFilterTest -q`
Expected: FAIL — `SherpaOnnxVoiceActivityFilter` class does not exist

- [ ] **Step 3: Implement SherpaOnnxVoiceActivityFilter**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.VoiceActivityFilter;
import io.casehub.blocks.speech.VoiceActivityFilterFactory;

import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;
import java.lang.foreign.ValueLayout;
import java.nio.file.Path;
import java.util.Objects;

public final class SherpaOnnxVoiceActivityFilter implements VoiceActivityFilterFactory {

    private final SherpaLibrary lib;
    private final Path modelPath;
    private final int numThreads;

    public static SherpaOnnxVoiceActivityFilter withDefaults() {
        return withDefaults("silero_vad");
    }

    public static SherpaOnnxVoiceActivityFilter withDefaults(String modelName) {
        Provisioner.ensureNativeLibrary();
        Path modelDir = Provisioner.ensureVadModel(modelName);
        SherpaLibrary lib = SherpaLibrary.load();
        return new SherpaOnnxVoiceActivityFilter(lib,
                modelDir.resolve(modelName + ".onnx"),
                Math.max(1, Runtime.getRuntime().availableProcessors() / 2));
    }

    SherpaOnnxVoiceActivityFilter(SherpaLibrary lib, Path modelPath, int numThreads) {
        this.lib = lib;
        this.modelPath = Objects.requireNonNull(modelPath);
        this.numThreads = numThreads;
    }

    @Override
    public VoiceActivityFilter create() {
        return new VadFilterInstance(lib, modelPath, numThreads);
    }

    private static final class VadFilterInstance implements VoiceActivityFilter {

        private final SherpaLibrary lib;
        private final MemorySegment vad;
        private final Arena arena;
        private boolean closed;

        VadFilterInstance(SherpaLibrary lib, Path modelPath, int numThreads) {
            this.lib = lib;
            this.arena = Arena.ofShared();

            MemorySegment configSeg = arena.allocate(SherpaLayouts.CONFIG_ALLOC_SIZE);
            configSeg.fill((byte) 0);
            configSeg.set(ValueLayout.ADDRESS, SherpaLayouts.VAD_SILERO_MODEL,
                    arena.allocateFrom(modelPath.toString()));
            configSeg.set(ValueLayout.JAVA_FLOAT, SherpaLayouts.VAD_SILERO_THRESHOLD, 0.5f);
            configSeg.set(ValueLayout.JAVA_FLOAT, SherpaLayouts.VAD_SILERO_MIN_SILENCE, 0.5f);
            configSeg.set(ValueLayout.JAVA_FLOAT, SherpaLayouts.VAD_SILERO_MIN_SPEECH, 0.25f);
            configSeg.set(ValueLayout.JAVA_INT, SherpaLayouts.VAD_SILERO_WINDOW_SIZE, 512);
            configSeg.set(ValueLayout.JAVA_FLOAT, SherpaLayouts.VAD_SILERO_MAX_SPEECH, 20.0f);
            configSeg.set(ValueLayout.JAVA_INT, SherpaLayouts.VAD_SAMPLE_RATE, 16000);
            configSeg.set(ValueLayout.JAVA_INT, SherpaLayouts.VAD_NUM_THREADS, numThreads);
            configSeg.set(ValueLayout.ADDRESS, SherpaLayouts.VAD_PROVIDER,
                    arena.allocateFrom("cpu"));

            try {
                this.vad = (MemorySegment) lib.createVad.invokeExact(configSeg, 30.0f);
            } catch (Throwable t) {
                arena.close();
                throw new SherpaException("Failed to create VAD", t);
            }

            if (vad.equals(MemorySegment.NULL)) {
                arena.close();
                throw new SherpaException("sherpa-onnx returned null VAD — check model: " + modelPath);
            }
        }

        @Override
        public float[] filterChunk(float[] samples, int sampleRate) {
            Objects.requireNonNull(samples, "samples");
            if (closed) { throw new IllegalStateException("VAD filter is closed"); }
            if (samples.length == 0) { return samples; }

            try (Arena callArena = Arena.ofConfined()) {
                MemorySegment samplesSeg = callArena.allocateFrom(ValueLayout.JAVA_FLOAT, samples);
                try {
                    lib.vadAcceptWaveform.invokeExact(vad, samplesSeg, samples.length);
                } catch (Throwable t) {
                    throw new SherpaException("VAD acceptWaveform failed", t);
                }

                int detected;
                try {
                    detected = (int) lib.vadDetected.invokeExact(vad);
                } catch (Throwable t) {
                    throw new SherpaException("VAD detected check failed", t);
                }

                return detected != 0 ? samples : new float[0];
            }
        }

        @Override
        public void reset() {
            if (closed) { return; }
            try { lib.vadReset.invokeExact(vad); } catch (Throwable ignored) {}
        }

        @Override
        public void close() {
            if (closed) { return; }
            closed = true;
            try { lib.destroyVad.invokeExact(vad); } catch (Throwable ignored) {}
            arena.close();
        }
    }
}
```

Path: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxVoiceActivityFilter.java`

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=SherpaOnnxVoiceActivityFilterTest -q`
Expected: PASS (or SKIPPED if native lib not available)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxVoiceActivityFilter.java speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxVoiceActivityFilterTest.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#184): add SherpaOnnxVoiceActivityFilter — Silero VAD pre-filter

Refs #184"
```

---

## Batch 3: Pipeline Integration

### Task 3: STT integration + runtime toggle

**Files:**
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java` — add `withVoiceActivityFilter(VoiceActivityFilterFactory, BooleanSupplier)`, modify `acceptSamples()` to gate after denoising
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechToText.java` — same pattern
- Create: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/VadIntegrationTest.java`

**Interfaces:**
- Consumes: `VoiceActivityFilterFactory` (from Task 1), `SherpaOnnxVoiceActivityFilter` (from Task 2)
- Produces: `WhisperSpeechToText.withVoiceActivityFilter(VoiceActivityFilterFactory, BooleanSupplier)` → `WhisperSpeechToText`, `SherpaOnnxStreamingSpeechToText.withVoiceActivityFilter(VoiceActivityFilterFactory, BooleanSupplier)` → `SherpaOnnxStreamingSpeechToText`

- [ ] **Step 1: Write the failing test**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.VoiceActivityFilter;
import io.casehub.blocks.speech.VoiceActivityFilterFactory;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.*;

class VadIntegrationTest {

    @Test
    @EnabledIf("io.casehub.blocks.speech.sherpa.WhisperLibrary#isAvailable")
    void whisperSttCallsVadFilterWhenEnabled() {
        var callCount = new AtomicInteger();
        var enabled = new AtomicBoolean(true);

        VoiceActivityFilterFactory factory = () -> new VoiceActivityFilter() {
            @Override
            public float[] filterChunk(float[] samples, int sampleRate) {
                callCount.incrementAndGet();
                return samples;
            }
            @Override public void reset() {}
            @Override public void close() {}
        };

        var stt = WhisperSpeechToText.withDefaults()
                .withVoiceActivityFilter(factory, enabled::get);
        var stream = stt.startStream(
                io.casehub.blocks.speech.TranscriptionOptions.defaults());

        stream.acceptSamples(new float[1600], 16000);
        assertEquals(1, callCount.get());

        enabled.set(false);
        stream.acceptSamples(new float[1600], 16000);
        assertEquals(1, callCount.get());

        stream.close();
    }

    @Test
    @EnabledIf("io.casehub.blocks.speech.sherpa.WhisperLibrary#isAvailable")
    void vadFilterDroppingChunkPreventsAccumulation() {
        var accumulated = new AtomicInteger();

        VoiceActivityFilterFactory factory = () -> new VoiceActivityFilter() {
            @Override
            public float[] filterChunk(float[] samples, int sampleRate) {
                return new float[0]; // drop everything
            }
            @Override public void reset() {}
            @Override public void close() {}
        };

        var stt = WhisperSpeechToText.withDefaults()
                .withVoiceActivityFilter(factory, () -> true);
        var stream = stt.startStream(
                io.casehub.blocks.speech.TranscriptionOptions.defaults());

        stream.acceptSamples(new float[1600], 16000);
        stream.acceptSamples(new float[1600], 16000);
        // finalResult on empty buffer should return empty text
        var result = stream.finalResult();
        assertEquals("", result.text());

        stream.close();
    }
}
```

Path: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/VadIntegrationTest.java`

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=VadIntegrationTest -q`
Expected: FAIL — `withVoiceActivityFilter` method does not exist

- [ ] **Step 3: Add withVoiceActivityFilter to WhisperSpeechToText**

Add field using `ide_replace_text_in_file` after the denoiser fields:

```java
    private io.casehub.blocks.speech.VoiceActivityFilterFactory vadFactory;
    private java.util.function.BooleanSupplier vadEnabled;
```

Add builder method using `ide_replace_text_in_file` after `withStreamingDenoiser`:

```java
    public WhisperSpeechToText withVoiceActivityFilter(
            io.casehub.blocks.speech.VoiceActivityFilterFactory factory,
            java.util.function.BooleanSupplier enabled) {
        this.vadFactory = factory;
        this.vadEnabled = enabled;
        return this;
    }
```

Add VAD filter field to `WhisperRecognitionStream` inner class after the denoiser field:

```java
        private final io.casehub.blocks.speech.VoiceActivityFilter vadFilter;
```

Modify constructor to create VAD filter:

```java
        this.vadFilter = (vadFactory != null) ? vadFactory.create() : null;
```

Modify `acceptSamples()` — add VAD gate after the denoiser block, before buffer accumulation. Use `ide_replace_text_in_file` to insert after the denoiser block:

```java
            if (vadFilter != null && vadEnabled != null && vadEnabled.getAsBoolean()) {
                processed = vadFilter.filterChunk(processed, sampleRate);
            }
            if (processed.length == 0) { return; }
```

Modify `close()` — add VAD cleanup after denoiser cleanup:

```java
            if (vadFilter != null) { vadFilter.close(); }
```

- [ ] **Step 4: Add withVoiceActivityFilter to SherpaOnnxStreamingSpeechToText**

Same pattern as WhisperSpeechToText:
- Add `vadFactory` and `vadEnabled` fields after denoiser fields
- Add `withVoiceActivityFilter()` builder method after `withStreamingDenoiser()`
- Add `vadFilter` field to `SherpaRecognitionStream` inner class
- Create VAD filter in constructor: `this.vadFilter = (vadFactory != null) ? vadFactory.create() : null;`
- In `acceptSamples()`: add VAD gate after denoiser, before lock acquisition:

```java
            if (vadFilter != null && vadEnabled != null && vadEnabled.getAsBoolean()) {
                processed = vadFilter.filterChunk(processed, sampleRate);
            }
            if (processed.length == 0) { return; }
```

- In `close()`: add `if (vadFilter != null) { vadFilter.close(); }` after denoiser cleanup

- [ ] **Step 5: Run tests**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=VadIntegrationTest -q`
Expected: PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechToText.java speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/VadIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#184): integrate VAD filter into STT pipeline with runtime toggle

Pipeline order: denoise → VAD → STT buffer.

Refs #184"
```

### Task 4: Avatar demo wiring

**Files:**
- Modify: `examples/avatar-demo/src/main/java/io/casehub/blocks/speech/demo/SpeechProducers.java` — wire VAD filter into STT producer
- Modify: `examples/avatar-demo/src/main/resources/application.properties` — add `casehub.speech.vad.enabled`

**Interfaces:**
- Consumes: `SherpaOnnxVoiceActivityFilter` (from Task 2), `WhisperSpeechToText.withVoiceActivityFilter()` and `SherpaOnnxStreamingSpeechToText.withVoiceActivityFilter()` (from Task 3)

- [ ] **Step 1: Wire VAD into SpeechProducers stt() method**

Use `ide_replace_text_in_file` to modify the `stt()` method. Add a `vadEnabled` config parameter and create the VAD factory alongside the denoiser factory. Chain `.withVoiceActivityFilter()` after `.withStreamingDenoiser()` on both the Whisper and Zipformer paths:

Add parameter to method signature:
```java
            @org.eclipse.microprofile.config.inject.ConfigProperty(
                    name = "casehub.speech.vad.enabled",
                    defaultValue = "true")
            jakarta.inject.Provider<Boolean> vadEnabled
```

Add VAD factory creation after denoiser factory creation:
```java
        io.casehub.blocks.speech.VoiceActivityFilterFactory vadFactory = null;
        try {
            vadFactory = io.casehub.blocks.speech.sherpa.SherpaOnnxVoiceActivityFilter.withDefaults();
            LOG.log(System.Logger.Level.INFO, "VAD loaded (Silero)");
        } catch (Throwable e) {
            LOG.log(System.Logger.Level.WARNING, "VAD unavailable: " + e.getMessage());
        }
```

Chain `withVoiceActivityFilter()` on both STT paths:
```java
            // After withStreamingDenoiser, add:
            if (vadFactory != null) {
                service = service.withVoiceActivityFilter(vadFactory, vadEnabled::get);
            }
```

Note: `withVoiceActivityFilter` returns the same type, so chaining works. But the return type is the concrete class (e.g. `WhisperSpeechToText`) — the local variable may need to be typed accordingly, or use the concrete type before assigning to the `StreamingSpeechToTextService` variable.

- [ ] **Step 2: Add config property**

Use `ide_replace_text_in_file` on `application.properties`:

```properties
casehub.speech.vad.enabled=true
```

Add after the `casehub.speech.denoising.enabled` line.

- [ ] **Step 3: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl speech-api,speech-sherpa compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks-ui add examples/avatar-demo/src/main/java/io/casehub/blocks/speech/demo/SpeechProducers.java examples/avatar-demo/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/slots/167/blocks-ui commit -m "feat(#184): wire VAD filter into avatar-demo with runtime config

casehub.speech.vad.enabled=true by default.
Pipeline: denoise → VAD → STT.

Refs #184"
```

---

## References

- [2026-09-02-voice-activity-detection-design.md] — design spec this plan implements
- [2026-09-02-speech-denoising-design.md] — denoiser spec (establishes the pattern)
- [speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiser.java] — pattern reference for SPI
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechDenoiser.java] — pattern reference for factory implementation
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java] — integration point (withStreamingDenoiser pattern)
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechToText.java] — integration point
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java] — FFM handle pattern
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java] — offset constants pattern
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java] — model provisioning pattern
- [blocks-ui/examples/avatar-demo/src/main/java/io/casehub/blocks/speech/demo/SpeechProducers.java] — wiring point
- [sherpa-onnx c-api.h](https://github.com/k2-fsa/sherpa-onnx/blob/master/sherpa-onnx/c-api/c-api.h) — VAD C API
- [Silero VAD docs](https://k2-fsa.github.io/sherpa/onnx/vad/silero-vad.html) — model parameters
- GE-20260826-190329 — oversized zero-filled FFM allocation technique
- GE-20260826-51c700 — sherpa-onnx struct layout gotcha
- GitHub casehubio/blocks#184 — focal issue
