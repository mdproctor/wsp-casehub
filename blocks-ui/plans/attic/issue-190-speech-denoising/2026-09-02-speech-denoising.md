# Speech Denoising Pre-processing Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #190 — Speech denoising pre-processing
**Issue group:** #190

**Goal:** Add optional, runtime-configurable speech denoising as a
pre-processing step before STT, using sherpa-onnx's DPDFNet (offline)
and GTCRN (online/streaming) models.

**Architecture:** Define `SpeechDenoiser` (offline) and
`StreamingSpeechDenoiser` (online) SPIs in `speech-api`. Implement via
FFM bindings in `speech-sherpa`. Inject into STT services compositionally
via `withDenoiser()`/`withStreamingDenoiser()` builder methods. Runtime
toggle via config property.

**Tech Stack:** Java 22+ FFM (Foreign Function & Memory), sherpa-onnx
v1.13.6 C API, DPDFNet baseline (offline, 16kHz), GTCRN simple (online,
16kHz).

## Global Constraints

- sherpa-onnx v1.13.6 native library required for integration tests
- FFM struct layouts use 4096-byte zero-filled allocation (GE-20260826-190329)
- `SherpaOnnxDenoisedAudio` has identical layout to `SherpaOnnxGeneratedAudio` — reuse `GENERATED_AUDIO` VarHandles from `SherpaLayouts`
- Online denoiser supports GTCRN only (not DPDFNet)
- All native tests gated with `@EnabledIf` on library availability

---

## Batch 1: SPI + FFM Foundation

### Task 1: SPI interfaces, FFM bindings, layout constants, model provisioning

**Files:**
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/SpeechDenoiser.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiser.java`
- Create: `speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiserFactory.java`
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java` — add 8 denoiser MethodHandle fields + downcall lookups
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java` — add denoiser config offset constants
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java` — add `DENOISER_MODEL_EXPECTED_FILES` map + `ensureDenoiserModel()` method

**Interfaces:**
- Produces: `SpeechDenoiser` — `float[] denoise(float[] samples, int sampleRate)`
- Produces: `StreamingSpeechDenoiser extends AutoCloseable` — `float[] processChunk(float[] samples, int sampleRate)`, `void reset()`, `void close()`
- Produces: `StreamingSpeechDenoiserFactory` — `StreamingSpeechDenoiser create()`
- Produces: SherpaLibrary fields `createOfflineDenoiser`, `destroyOfflineDenoiser`, `offlineDenoiserRun`, `destroyDenoisedAudio`, `createOnlineDenoiser`, `destroyOnlineDenoiser`, `onlineDenoiserRun`, `onlineDenoiserReset`
- Produces: SherpaLayouts constants `DENOISER_GTCRN_MODEL`, `DENOISER_NUM_THREADS`, `DENOISER_DEBUG`, `DENOISER_PROVIDER`, `DENOISER_DPDFNET_MODEL`
- Produces: `Provisioner.ensureDenoiserModel(String modelName)` → `Path`

- [ ] **Step 1: Create SpeechDenoiser interface**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech;

public interface SpeechDenoiser {
    float[] denoise(float[] samples, int sampleRate);
}
```

Path: `speech-api/src/main/java/io/casehub/blocks/speech/SpeechDenoiser.java`

- [ ] **Step 2: Create StreamingSpeechDenoiser interface**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech;

public interface StreamingSpeechDenoiser extends AutoCloseable {
    float[] processChunk(float[] samples, int sampleRate);
    void reset();
    void close();
}
```

Path: `speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiser.java`

- [ ] **Step 3: Create StreamingSpeechDenoiserFactory interface**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech;

public interface StreamingSpeechDenoiserFactory {
    StreamingSpeechDenoiser create();
}
```

Path: `speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiserFactory.java`

- [ ] **Step 4: Add denoiser config offset constants to SherpaLayouts**

Use `ide_insert_member` to add before the closing `}` of `SherpaLayouts`:

```java
// === SherpaOnnxOfflineSpeechDenoiserConfig / SherpaOnnxOnlineSpeechDenoiserConfig offsets ===
// Both configs start with model sub-struct at offset 0.
// Offline: gtcrn(8) + num_threads(4) + debug(4) + provider(8) + dpdfnet(8) = 32 bytes
// Online:  gtcrn(8) + num_threads(4) + debug(4) + provider(8) = 24 bytes
static final long DENOISER_GTCRN_MODEL   = 0;
static final long DENOISER_NUM_THREADS   = 8;
static final long DENOISER_DEBUG         = 12;
static final long DENOISER_PROVIDER      = 16;
static final long DENOISER_DPDFNET_MODEL = 24;  // offline only
```

- [ ] **Step 5: Add 8 denoiser MethodHandle fields to SherpaLibrary**

Use `ide_insert_member` to add fields after `onlinePunctuationFreeText`:

```java
// Offline speech denoiser handles
final MethodHandle createOfflineDenoiser;
final MethodHandle destroyOfflineDenoiser;
final MethodHandle offlineDenoiserRun;
final MethodHandle destroyDenoisedAudio;
// Online speech denoiser handles
final MethodHandle createOnlineDenoiser;
final MethodHandle destroyOnlineDenoiser;
final MethodHandle onlineDenoiserRun;
final MethodHandle onlineDenoiserReset;
```

- [ ] **Step 6: Add downcall lookups in SherpaLibrary constructor**

Use `ide_edit_member` on the constructor to append after the punctuation handles block:

```java
// Offline speech denoiser handles
createOfflineDenoiser = downcall(linker, "SherpaOnnxCreateOfflineSpeechDenoiser",
        FunctionDescriptor.of(ADDRESS, ADDRESS));
destroyOfflineDenoiser = downcall(linker, "SherpaOnnxDestroyOfflineSpeechDenoiser",
        FunctionDescriptor.ofVoid(ADDRESS));
offlineDenoiserRun = downcall(linker, "SherpaOnnxOfflineSpeechDenoiserRun",
        FunctionDescriptor.of(ADDRESS, ADDRESS, ADDRESS, JAVA_INT, JAVA_INT));
destroyDenoisedAudio = downcall(linker, "SherpaOnnxDestroyDenoisedAudio",
        FunctionDescriptor.ofVoid(ADDRESS));
// Online speech denoiser handles
createOnlineDenoiser = downcall(linker, "SherpaOnnxCreateOnlineSpeechDenoiser",
        FunctionDescriptor.of(ADDRESS, ADDRESS));
destroyOnlineDenoiser = downcall(linker, "SherpaOnnxDestroyOnlineSpeechDenoiser",
        FunctionDescriptor.ofVoid(ADDRESS));
onlineDenoiserRun = downcall(linker, "SherpaOnnxOnlineSpeechDenoiserRun",
        FunctionDescriptor.of(ADDRESS, ADDRESS, ADDRESS, JAVA_INT, JAVA_INT));
onlineDenoiserReset = downcall(linker, "SherpaOnnxOnlineSpeechDenoiserReset",
        FunctionDescriptor.ofVoid(ADDRESS));
```

- [ ] **Step 7: Add model provisioning to Provisioner**

Use `ide_insert_member` to add after `KOKORO_MODEL_EXPECTED_FILES`:

```java
private static final Map<String, String> DENOISER_MODEL_EXPECTED_FILES = Map.of(
        "dpdfnet_baseline", "dpdfnet_baseline.onnx",
        "gtcrn_simple", "gtcrn_simple.onnx"
);
```

Add `denoiserModelDir` method:

```java
static Path denoiserModelDir(String modelName) {
    return cacheBaseDir().resolve("models").resolve("sherpa-onnx").resolve(modelName);
}
```

Add `denoiserModelUrl` method:

```java
static String denoiserModelUrl(String modelName) {
    return baseUrl() + "speech-enhancement-models/" + modelName + ".onnx";
}
```

Add `ensureDenoiserModel` method:

```java
static Path ensureDenoiserModel(String modelName) {
    String expectedFile = DENOISER_MODEL_EXPECTED_FILES.get(modelName);
    if (expectedFile == null) {
        throw new SherpaException(
                "Unknown denoiser model: " + modelName
                + ". Known models: " + DENOISER_MODEL_EXPECTED_FILES.keySet());
    }
    Path targetDir = denoiserModelDir(modelName);

    if (Files.isDirectory(targetDir) && Files.exists(targetDir.resolve(expectedFile))) {
        return targetDir;
    }

    synchronized (MODEL_LOCK) {
        if (Files.isDirectory(targetDir) && Files.exists(targetDir.resolve(expectedFile))) {
            return targetDir;
        }
        try {
            Files.createDirectories(targetDir);
            String url = denoiserModelUrl(modelName);
            LOG.log(System.Logger.Level.INFO, "Downloading {0}...", url);
            Path downloaded = downloadWithRetry(url, targetDir, 3);
            Path dest = targetDir.resolve(expectedFile);
            if (!downloaded.getFileName().toString().equals(expectedFile)) {
                java.nio.file.Files.move(downloaded, dest, StandardCopyOption.REPLACE_EXISTING);
            }
        } catch (IOException e) {
            throw new SherpaException("Failed to download denoiser model: " + modelName
                                      + ". Download manually to " + targetDir, e);
        }
        return targetDir;
    }
}
```

Note: denoiser models are single `.onnx` files (not tarballs), so we download directly to the target directory rather than extracting an archive.

- [ ] **Step 8: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl speech-api compile -q && /opt/homebrew/bin/mvn -pl speech-sherpa compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add speech-api/src/main/java/io/casehub/blocks/speech/SpeechDenoiser.java speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiser.java speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechDenoiserFactory.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#190): add SpeechDenoiser SPIs, FFM bindings, and model provisioning

Refs #190"
```

---

## Batch 2: Denoiser Implementations

### Task 2: Offline denoiser — SherpaOnnxSpeechDenoiser

**Files:**
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechDenoiser.java`
- Create: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechDenoiserTest.java`

**Interfaces:**
- Consumes: `SpeechDenoiser` (from Task 1), `SherpaLibrary` fields (`createOfflineDenoiser`, `offlineDenoiserRun`, `destroyDenoisedAudio`, `destroyOfflineDenoiser`), `SherpaLayouts` constants (`DENOISER_DPDFNET_MODEL`, `DENOISER_NUM_THREADS`, `DENOISER_PROVIDER`), `Provisioner.ensureDenoiserModel(String)`
- Produces: `SherpaOnnxSpeechDenoiser` — `static withDefaults()`, `static withDefaults(String modelName)`, `float[] denoise(float[] samples, int sampleRate)`, implements `AutoCloseable`

- [ ] **Step 1: Write the failing test**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.SpeechDenoiser;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import static org.junit.jupiter.api.Assertions.*;

@EnabledIf("io.casehub.blocks.speech.sherpa.SherpaLibrary#isAvailable")
class SherpaOnnxSpeechDenoiserTest {

    @Test
    void denoisesAudioAndReturnsSameLengthOutput() {
        SpeechDenoiser denoiser = SherpaOnnxSpeechDenoiser.withDefaults();
        float[] noisy = new float[16000]; // 1 second of silence at 16kHz
        for (int i = 0; i < noisy.length; i++) {
            noisy[i] = (float) (Math.sin(2 * Math.PI * 440 * i / 16000) * 0.5
                                + Math.random() * 0.1);
        }

        float[] denoised = denoiser.denoise(noisy, 16000);

        assertNotNull(denoised);
        assertEquals(noisy.length, denoised.length);
        assertNotEquals(0, denoised.length);
    }

    @Test
    void implementsSpeechDenoiserInterface() {
        SpeechDenoiser denoiser = SherpaOnnxSpeechDenoiser.withDefaults();
        assertInstanceOf(SpeechDenoiser.class, denoiser);
    }
}
```

Path: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechDenoiserTest.java`

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=SherpaOnnxSpeechDenoiserTest -q`
Expected: FAIL — `SherpaOnnxSpeechDenoiser` class does not exist

- [ ] **Step 3: Implement SherpaOnnxSpeechDenoiser**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.SpeechDenoiser;

import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;
import java.lang.foreign.ValueLayout;
import java.nio.file.Path;
import java.util.Objects;

public final class SherpaOnnxSpeechDenoiser implements SpeechDenoiser, AutoCloseable {

    private final SherpaLibrary lib;
    private final MemorySegment denoiser;
    private final Arena denoiserArena;

    public static SherpaOnnxSpeechDenoiser withDefaults() {
        return withDefaults("dpdfnet_baseline");
    }

    public static SherpaOnnxSpeechDenoiser withDefaults(String modelName) {
        Provisioner.ensureNativeLibrary();
        Path modelDir = Provisioner.ensureDenoiserModel(modelName);
        SherpaLibrary lib = SherpaLibrary.load();
        return new SherpaOnnxSpeechDenoiser(lib, modelDir.resolve(modelName + ".onnx"));
    }

    SherpaOnnxSpeechDenoiser(SherpaLibrary lib, Path modelPath) {
        this.lib = lib;
        this.denoiserArena = Arena.ofShared();

        MemorySegment configSeg = denoiserArena.allocate(SherpaLayouts.CONFIG_ALLOC_SIZE);
        configSeg.fill((byte) 0);
        configSeg.set(ValueLayout.ADDRESS, SherpaLayouts.DENOISER_DPDFNET_MODEL,
                denoiserArena.allocateFrom(modelPath.toString()));
        configSeg.set(ValueLayout.JAVA_INT, SherpaLayouts.DENOISER_NUM_THREADS,
                Math.max(1, Runtime.getRuntime().availableProcessors() / 2));
        configSeg.set(ValueLayout.ADDRESS, SherpaLayouts.DENOISER_PROVIDER,
                denoiserArena.allocateFrom("cpu"));

        try {
            this.denoiser = (MemorySegment) lib.createOfflineDenoiser.invokeExact(configSeg);
        } catch (Throwable t) {
            denoiserArena.close();
            throw new SherpaException("Failed to create offline speech denoiser", t);
        }

        if (denoiser.equals(MemorySegment.NULL)) {
            denoiserArena.close();
            throw new SherpaException("sherpa-onnx returned null offline denoiser — check model: " + modelPath);
        }
    }

    @Override
    public float[] denoise(float[] samples, int sampleRate) {
        Objects.requireNonNull(samples, "samples");
        if (samples.length == 0) { return samples; }

        try (Arena arena = Arena.ofConfined()) {
            MemorySegment samplesSeg = arena.allocateFrom(ValueLayout.JAVA_FLOAT, samples);

            MemorySegment resultPtr;
            try {
                resultPtr = (MemorySegment) lib.offlineDenoiserRun.invokeExact(
                        denoiser, samplesSeg, samples.length, sampleRate);
            } catch (Throwable t) {
                throw new SherpaException("Offline denoiser run failed", t);
            }

            try {
                MemorySegment result = resultPtr.reinterpret(SherpaLayouts.GENERATED_AUDIO.byteSize());
                int n = (int) SherpaLayouts.AUDIO_N.get(result, 0L);
                MemorySegment denoisedPtr = (MemorySegment) SherpaLayouts.AUDIO_SAMPLES.get(result, 0L);
                return denoisedPtr
                        .reinterpret((long) n * ValueLayout.JAVA_FLOAT.byteSize())
                        .toArray(ValueLayout.JAVA_FLOAT);
            } finally {
                try { lib.destroyDenoisedAudio.invokeExact(resultPtr); } catch (Throwable ignored) {}
            }
        }
    }

    @Override
    public void close() {
        try { lib.destroyOfflineDenoiser.invokeExact(denoiser); } catch (Throwable ignored) {}
        denoiserArena.close();
    }
}
```

Path: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechDenoiser.java`

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=SherpaOnnxSpeechDenoiserTest -q`
Expected: PASS (or SKIPPED if native lib not available)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechDenoiser.java speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechDenoiserTest.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#190): add SherpaOnnxSpeechDenoiser — offline DPDFNet denoiser

Refs #190"
```

### Task 3: Online streaming denoiser — SherpaOnnxStreamingSpeechDenoiser

**Files:**
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechDenoiser.java`
- Create: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechDenoiserTest.java`

**Interfaces:**
- Consumes: `StreamingSpeechDenoiser`, `StreamingSpeechDenoiserFactory` (from Task 1), `SherpaLibrary` fields (`createOnlineDenoiser`, `onlineDenoiserRun`, `onlineDenoiserReset`, `destroyOnlineDenoiser`, `destroyDenoisedAudio`), `SherpaLayouts` constants (`DENOISER_GTCRN_MODEL`, `DENOISER_NUM_THREADS`, `DENOISER_PROVIDER`), `Provisioner.ensureDenoiserModel(String)`
- Produces: `SherpaOnnxStreamingSpeechDenoiser` — implements `StreamingSpeechDenoiserFactory`, `static withDefaults()`, `static withDefaults(String modelName)`, `StreamingSpeechDenoiser create()`

- [ ] **Step 1: Write the failing test**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.StreamingSpeechDenoiser;
import io.casehub.blocks.speech.StreamingSpeechDenoiserFactory;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.condition.EnabledIf;

import static org.junit.jupiter.api.Assertions.*;

@EnabledIf("io.casehub.blocks.speech.sherpa.SherpaLibrary#isAvailable")
class SherpaOnnxStreamingSpeechDenoiserTest {

    @Test
    void factoryCreatesDenoiserInstances() {
        StreamingSpeechDenoiserFactory factory = SherpaOnnxStreamingSpeechDenoiser.withDefaults();
        try (StreamingSpeechDenoiser d1 = factory.create();
             StreamingSpeechDenoiser d2 = factory.create()) {
            assertNotNull(d1);
            assertNotNull(d2);
            assertNotSame(d1, d2);
        }
    }

    @Test
    void processChunkReturnsDenoisedAudio() {
        StreamingSpeechDenoiserFactory factory = SherpaOnnxStreamingSpeechDenoiser.withDefaults();
        try (StreamingSpeechDenoiser denoiser = factory.create()) {
            float[] chunk = new float[4096];
            for (int i = 0; i < chunk.length; i++) {
                chunk[i] = (float) (Math.sin(2 * Math.PI * 440 * i / 16000) * 0.5
                                    + Math.random() * 0.1);
            }

            float[] denoised = denoiser.processChunk(chunk, 16000);
            assertNotNull(denoised);
            assertTrue(denoised.length > 0);
        }
    }

    @Test
    void resetClearsInternalState() {
        StreamingSpeechDenoiserFactory factory = SherpaOnnxStreamingSpeechDenoiser.withDefaults();
        try (StreamingSpeechDenoiser denoiser = factory.create()) {
            float[] chunk = new float[4096];
            denoiser.processChunk(chunk, 16000);
            assertDoesNotThrow(denoiser::reset);
            float[] afterReset = denoiser.processChunk(chunk, 16000);
            assertNotNull(afterReset);
        }
    }
}
```

Path: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechDenoiserTest.java`

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=SherpaOnnxStreamingSpeechDenoiserTest -q`
Expected: FAIL — `SherpaOnnxStreamingSpeechDenoiser` class does not exist

- [ ] **Step 3: Implement SherpaOnnxStreamingSpeechDenoiser**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.StreamingSpeechDenoiser;
import io.casehub.blocks.speech.StreamingSpeechDenoiserFactory;

import java.lang.foreign.Arena;
import java.lang.foreign.MemorySegment;
import java.lang.foreign.ValueLayout;
import java.nio.file.Path;
import java.util.Objects;

public final class SherpaOnnxStreamingSpeechDenoiser implements StreamingSpeechDenoiserFactory {

    private final SherpaLibrary lib;
    private final Path modelPath;
    private final int numThreads;

    public static SherpaOnnxStreamingSpeechDenoiser withDefaults() {
        return withDefaults("gtcrn_simple");
    }

    public static SherpaOnnxStreamingSpeechDenoiser withDefaults(String modelName) {
        Provisioner.ensureNativeLibrary();
        Path modelDir = Provisioner.ensureDenoiserModel(modelName);
        SherpaLibrary lib = SherpaLibrary.load();
        return new SherpaOnnxStreamingSpeechDenoiser(lib,
                modelDir.resolve(modelName + ".onnx"),
                Math.max(1, Runtime.getRuntime().availableProcessors() / 2));
    }

    SherpaOnnxStreamingSpeechDenoiser(SherpaLibrary lib, Path modelPath, int numThreads) {
        this.lib = lib;
        this.modelPath = Objects.requireNonNull(modelPath);
        this.numThreads = numThreads;
    }

    @Override
    public StreamingSpeechDenoiser create() {
        return new OnlineDenoiserInstance(lib, modelPath, numThreads);
    }

    private static final class OnlineDenoiserInstance implements StreamingSpeechDenoiser {

        private final SherpaLibrary lib;
        private final MemorySegment denoiser;
        private final Arena arena;
        private boolean closed;

        OnlineDenoiserInstance(SherpaLibrary lib, Path modelPath, int numThreads) {
            this.lib = lib;
            this.arena = Arena.ofShared();

            MemorySegment configSeg = arena.allocate(SherpaLayouts.CONFIG_ALLOC_SIZE);
            configSeg.fill((byte) 0);
            configSeg.set(ValueLayout.ADDRESS, SherpaLayouts.DENOISER_GTCRN_MODEL,
                    arena.allocateFrom(modelPath.toString()));
            configSeg.set(ValueLayout.JAVA_INT, SherpaLayouts.DENOISER_NUM_THREADS, numThreads);
            configSeg.set(ValueLayout.ADDRESS, SherpaLayouts.DENOISER_PROVIDER,
                    arena.allocateFrom("cpu"));

            try {
                this.denoiser = (MemorySegment) lib.createOnlineDenoiser.invokeExact(configSeg);
            } catch (Throwable t) {
                arena.close();
                throw new SherpaException("Failed to create online speech denoiser", t);
            }

            if (denoiser.equals(MemorySegment.NULL)) {
                arena.close();
                throw new SherpaException("sherpa-onnx returned null online denoiser — check model: " + modelPath);
            }
        }

        @Override
        public float[] processChunk(float[] samples, int sampleRate) {
            Objects.requireNonNull(samples, "samples");
            if (closed) { throw new IllegalStateException("Denoiser is closed"); }
            if (samples.length == 0) { return samples; }

            try (Arena callArena = Arena.ofConfined()) {
                MemorySegment samplesSeg = callArena.allocateFrom(ValueLayout.JAVA_FLOAT, samples);

                MemorySegment resultPtr;
                try {
                    resultPtr = (MemorySegment) lib.onlineDenoiserRun.invokeExact(
                            denoiser, samplesSeg, samples.length, sampleRate);
                } catch (Throwable t) {
                    throw new SherpaException("Online denoiser run failed", t);
                }

                try {
                    MemorySegment result = resultPtr.reinterpret(SherpaLayouts.GENERATED_AUDIO.byteSize());
                    int n = (int) SherpaLayouts.AUDIO_N.get(result, 0L);
                    if (n == 0) { return new float[0]; }
                    MemorySegment denoisedPtr = (MemorySegment) SherpaLayouts.AUDIO_SAMPLES.get(result, 0L);
                    return denoisedPtr
                            .reinterpret((long) n * ValueLayout.JAVA_FLOAT.byteSize())
                            .toArray(ValueLayout.JAVA_FLOAT);
                } finally {
                    try { lib.destroyDenoisedAudio.invokeExact(resultPtr); } catch (Throwable ignored) {}
                }
            }
        }

        @Override
        public void reset() {
            if (closed) { return; }
            try { lib.onlineDenoiserReset.invokeExact(denoiser); } catch (Throwable ignored) {}
        }

        @Override
        public void close() {
            if (closed) { return; }
            closed = true;
            try { lib.destroyOnlineDenoiser.invokeExact(denoiser); } catch (Throwable ignored) {}
            arena.close();
        }
    }
}
```

Path: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechDenoiser.java`

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=SherpaOnnxStreamingSpeechDenoiserTest -q`
Expected: PASS (or SKIPPED if native lib not available)

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechDenoiser.java speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechDenoiserTest.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#190): add SherpaOnnxStreamingSpeechDenoiser — online GTCRN denoiser

Refs #190"
```

---

## Batch 3: Pipeline Integration

### Task 4: STT integration + runtime toggle + avatar demo wiring

**Files:**
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java` — add `withStreamingDenoiser(StreamingSpeechDenoiserFactory, BooleanSupplier)` method, modify `WhisperRecognitionStream` to denoise chunks in `acceptSamples()`
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechToText.java` — add `withStreamingDenoiser()`, modify `SherpaRecognitionStream.acceptSamples()` to denoise chunks
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechToText.java` — add `withDenoiser(SpeechDenoiser, BooleanSupplier)`, denoise WAV samples before recognition
- Create: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/DenoiserIntegrationTest.java`
- Modify: `examples/avatar-demo/src/main/java/io/casehub/blocks/speech/demo/SpeechProducers.java` — wire denoiser into STT producer
- Modify: `examples/avatar-demo/src/main/resources/application.properties` — add `casehub.speech.denoising.enabled` config

**Interfaces:**
- Consumes: `StreamingSpeechDenoiserFactory` (from Task 1), `SherpaOnnxStreamingSpeechDenoiser` (from Task 3), `SpeechDenoiser` (from Task 1), `SherpaOnnxSpeechDenoiser` (from Task 2)
- Produces: `WhisperSpeechToText.withStreamingDenoiser(StreamingSpeechDenoiserFactory, BooleanSupplier)` → `WhisperSpeechToText`, `SherpaOnnxStreamingSpeechToText.withStreamingDenoiser(StreamingSpeechDenoiserFactory, BooleanSupplier)` → `SherpaOnnxStreamingSpeechToText`, `SherpaOnnxSpeechToText.withDenoiser(SpeechDenoiser, BooleanSupplier)` → `SherpaOnnxSpeechToText`

- [ ] **Step 1: Write the failing test for runtime toggle**

Use `ide_create_file`:

```java
package io.casehub.blocks.speech.sherpa;

import io.casehub.blocks.speech.StreamingSpeechDenoiser;
import io.casehub.blocks.speech.StreamingSpeechDenoiserFactory;
import org.junit.jupiter.api.Test;

import java.util.concurrent.atomic.AtomicBoolean;
import java.util.concurrent.atomic.AtomicInteger;

import static org.junit.jupiter.api.Assertions.*;

class DenoiserIntegrationTest {

    @Test
    void whisperSttCallsDenoiserWhenEnabled() {
        var callCount = new AtomicInteger();
        var enabled = new AtomicBoolean(true);

        StreamingSpeechDenoiserFactory factory = () -> new StreamingSpeechDenoiser() {
            @Override
            public float[] processChunk(float[] samples, int sampleRate) {
                callCount.incrementAndGet();
                return samples;
            }
            @Override public void reset() {}
            @Override public void close() {}
        };

        var stt = WhisperSpeechToText.withDefaults()
                .withStreamingDenoiser(factory, enabled::get);
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
    void sttWithoutDenoiserPassesThroughUnchanged() {
        var stt = WhisperSpeechToText.withDefaults();
        var stream = stt.startStream(
                io.casehub.blocks.speech.TranscriptionOptions.defaults());
        assertDoesNotThrow(() -> stream.acceptSamples(new float[1600], 16000));
        stream.close();
    }
}
```

Path: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/DenoiserIntegrationTest.java`

- [ ] **Step 2: Run test to verify it fails**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=DenoiserIntegrationTest -q`
Expected: FAIL — `withStreamingDenoiser` method does not exist on `WhisperSpeechToText`

- [ ] **Step 3: Add withStreamingDenoiser to WhisperSpeechToText**

Use `ide_insert_member` to add fields after `inferenceLock`:

```java
private final @Nullable StreamingSpeechDenoiserFactory denoiserFactory;
private final @Nullable java.util.function.BooleanSupplier denoiserEnabled;
```

Add import for `StreamingSpeechDenoiserFactory`:
```java
import io.casehub.blocks.speech.StreamingSpeechDenoiserFactory;
```

Use `ide_insert_member` to add the builder method:

```java
public WhisperSpeechToText withStreamingDenoiser(
        StreamingSpeechDenoiserFactory factory,
        java.util.function.BooleanSupplier enabled) {
    var copy = new WhisperSpeechToText(this.lib, this.ctx, this.ctxArena);
    copy.denoiserFactory = factory;
    copy.denoiserEnabled = enabled;
    return copy;
}
```

Note: This requires a copy-constructor approach. Since `WhisperSpeechToText` manages native state (`ctx`, `ctxArena`), the builder returns a new wrapper sharing the same underlying native context. The `denoiserFactory` and `denoiserEnabled` fields default to `null` in the existing constructors.

Modify `WhisperRecognitionStream` inner class:
- Add a `@Nullable StreamingSpeechDenoiser denoiser` field, initialized from `denoiserFactory.create()` if factory is non-null
- In `acceptSamples()`: if `denoiser != null && denoiserEnabled.getAsBoolean()`, call `denoiser.processChunk(samples, sampleRate)` and use the result instead of the raw samples
- In `close()`: close the denoiser if non-null

Use `ide_edit_member` on `WhisperRecognitionStream` constructor to add:

```java
this.denoiser = (denoiserFactory != null) ? denoiserFactory.create() : null;
```

Use `ide_edit_member` on `acceptSamples` to wrap the sample handling:

```java
@Override
public void acceptSamples(float[] samples, int sampleRate) {
    if (closed) return;
    if (sampleRate != 16000) {
        throw new IllegalArgumentException("Whisper requires 16kHz audio, got " + sampleRate);
    }
    float[] processed = samples;
    if (denoiser != null && denoiserEnabled != null && denoiserEnabled.getAsBoolean()) {
        processed = denoiser.processChunk(samples, sampleRate);
    }
    // ... existing buffer accumulation logic using 'processed' instead of 'samples' ...
}
```

Use `ide_edit_member` on `close` to add denoiser cleanup:

```java
@Override
public void close() {
    closed = true;
    buffer = null;
    sampleCount = 0;
    if (denoiser != null) { denoiser.close(); }
}
```

- [ ] **Step 4: Run test to verify it passes**

Run: `/opt/homebrew/bin/mvn -pl speech-sherpa test -Dtest=DenoiserIntegrationTest -q`
Expected: PASS

- [ ] **Step 5: Add withStreamingDenoiser to SherpaOnnxStreamingSpeechToText**

Same pattern as WhisperSpeechToText. Use `ide_insert_member` to add fields and builder method. Modify `SherpaRecognitionStream.acceptSamples()` to denoise chunks before feeding the native recognizer.

- [ ] **Step 6: Add withDenoiser to SherpaOnnxSpeechToText (file-based)**

Use `ide_insert_member` to add fields:

```java
private @Nullable SpeechDenoiser denoiser;
private @Nullable java.util.function.BooleanSupplier denoiserEnabled;
```

Add builder method:

```java
public SherpaOnnxSpeechToText withDenoiser(
        SpeechDenoiser denoiser,
        java.util.function.BooleanSupplier enabled) {
    this.denoiser = denoiser;
    this.denoiserEnabled = enabled;
    return this;
}
```

In `transcribe()`, after `WavReader.read(audioFile)` and before building the recognizer config, add:

```java
float[] samples = wav.samples();
if (denoiser != null && denoiserEnabled != null && denoiserEnabled.getAsBoolean()) {
    samples = denoiser.denoise(samples, wav.sampleRate());
}
```

Use the `samples` variable instead of `wav.samples()` in the subsequent `MemorySegment.copy` call.

- [ ] **Step 7: Wire denoiser into avatar-demo SpeechProducers**

Use `ide_edit_member` on the `stt()` method in `SpeechProducers.java` to add denoiser wiring:

```java
@Produces
@jakarta.inject.Singleton
io.casehub.blocks.speech.StreamingSpeechToTextService stt(
        @org.eclipse.microprofile.config.inject.ConfigProperty(
                name = "casehub.speech.denoising.enabled",
                defaultValue = "true")
        jakarta.inject.Provider<Boolean> denoisingEnabled) {
    io.casehub.blocks.speech.StreamingSpeechDenoiserFactory denoiserFactory = null;
    try {
        denoiserFactory = io.casehub.blocks.speech.sherpa.SherpaOnnxStreamingSpeechDenoiser.withDefaults();
        LOG.log(System.Logger.Level.INFO, "Speech denoiser loaded (GTCRN)");
    } catch (Throwable e) {
        LOG.log(System.Logger.Level.WARNING, "Speech denoiser unavailable: " + e.getMessage());
    }

    io.casehub.blocks.speech.StreamingSpeechToTextService service;
    try {
        io.casehub.blocks.speech.sherpa.WhisperLibrary.load();
        LOG.log(System.Logger.Level.INFO, "Using WhisperSpeechToText");
        whisperActive = true;
        var whisper = io.casehub.blocks.speech.sherpa.WhisperSpeechToText.withDefaults();
        service = denoiserFactory != null
                ? whisper.withStreamingDenoiser(denoiserFactory, denoisingEnabled::get)
                : whisper;
    } catch (Throwable e) {
        LOG.log(System.Logger.Level.WARNING, "Whisper unavailable, falling back to Zipformer: "
                + e.getClass().getSimpleName() + ": " + e.getMessage(), e);
        var zipformer = io.casehub.blocks.speech.sherpa.SherpaOnnxStreamingSpeechToText.withDefaults();
        service = denoiserFactory != null
                ? zipformer.withStreamingDenoiser(denoiserFactory, denoisingEnabled::get)
                : zipformer;
    }
    return service;
}
```

- [ ] **Step 8: Add config property to application.properties**

Add to `examples/avatar-demo/src/main/resources/application.properties`:

```properties
casehub.speech.denoising.enabled=true
```

- [ ] **Step 9: Verify compilation**

Run: `/opt/homebrew/bin/mvn -pl speech-api,speech-sherpa,speech-ws compile -q`
Expected: BUILD SUCCESS

- [ ] **Step 10: Run all speech tests**

Run: `/opt/homebrew/bin/mvn -pl speech-api,speech-sherpa test -q`
Expected: All tests PASS

- [ ] **Step 11: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxStreamingSpeechToText.java speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechToText.java speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/DenoiserIntegrationTest.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#190): integrate denoiser into STT pipeline with runtime toggle

Refs #190"
```

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks-ui add examples/avatar-demo/src/main/java/io/casehub/blocks/speech/demo/SpeechProducers.java examples/avatar-demo/src/main/resources/application.properties
git -C /Users/mdproctor/claude/casehub/slots/167/blocks-ui commit -m "feat(#190): wire speech denoiser into avatar-demo with runtime config

casehub.speech.denoising.enabled=true by default.

Refs #190"
```

---

## References

- [2026-09-02-speech-denoising-design.md] — design spec this plan implements
- [speech-api/src/main/java/io/casehub/blocks/speech/SpeechToTextService.java] — existing offline STT SPI
- [speech-api/src/main/java/io/casehub/blocks/speech/StreamingSpeechToTextService.java] — existing streaming STT SPI
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java] — FFM handle loading pattern
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLayouts.java] — struct offset constants
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java] — model download pattern
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/WhisperSpeechToText.java:70-169] — streaming STT with buffer accumulation
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechToText.java] — file-based STT
- [speech-ws/src/main/java/io/casehub/blocks/speech/ws/SpeechSession.java:135-138] — audio handling (unchanged by this plan)
- [blocks-ui/examples/avatar-demo/src/main/java/io/casehub/blocks/speech/demo/SpeechProducers.java:191-201] — STT producer wiring point
- GE-20260826-190329 — oversized zero-filled FFM allocation technique
- GE-20260826-51c700 — sherpa-onnx struct layout must match all nested sub-configs
- GE-20260803-e363e6 — ONNX Runtime SIGSEGV with concurrent access
- [sherpa-onnx c-api.h](https://github.com/k2-fsa/sherpa-onnx/blob/master/sherpa-onnx/c-api/c-api.h) — native API declarations
- GitHub casehubio/blocks#190 — focal issue
