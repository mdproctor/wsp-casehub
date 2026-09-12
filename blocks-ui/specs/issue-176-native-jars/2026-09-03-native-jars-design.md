# Platform-Specific Maven JARs for Native Lib Bundling

**Issue:** casehubio/blocks#176
**Parent:** casehubio/blocks#174 (Zero-Install Speech Experience)
**Date:** 2026-09-03

## Summary

Publish platform-specific Maven JAR artifacts containing sherpa-onnx and onnxruntime
native libraries. Consumers add one `runtime` dependency for their platform — zero
manual native lib installation. Pattern follows Netty native transports.

## Architecture

### Module Structure

5 new Maven modules under `speech-sherpa/` in the blocks repo:

```
speech-sherpa/
├── speech-sherpa-native-osx-arm64/
├── speech-sherpa-native-osx-x64/
├── speech-sherpa-native-linux-x64/
├── speech-sherpa-native-linux-arm64/
├── speech-sherpa-native-win-x64/
```

Each module is a standard Maven JAR containing native libs at:
```
META-INF/native/sherpa-onnx/<version>/<platform>/
  libsherpa-onnx-c-api.dylib  (or .so / .dll)
  libonnxruntime.dylib         (or .so / .dll)
```

Version and platform in the resource path prevents classpath conflicts when
multiple versions coexist.

### Consumer Usage

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>speech-sherpa-native-osx-arm64</artifactId>
    <version>${casehub.version}</version>
    <scope>runtime</scope>
</dependency>
```

### Bundling

Each platform JAR bundles both `libsherpa-onnx-c-api` and `libonnxruntime`
together (~31MB per JAR). These libs are tightly coupled — sherpa-onnx links
against a specific onnxruntime version — so separate JARs would add dependency
management burden for a version-locked pair.

## Runtime Extraction and Loading

### SherpaLibrary Loading Tiers (Updated)

```
Tier 1:   System library path (SymbolLookup.libraryLookup by name)
Tier 1.5: Classpath JAR extraction (NEW)
Tier 2:   Local cache (~/.casehub/native/sherpa-onnx/<version>/<platform>/)
Tier 3:   Auto-download from GitHub releases
```

### Tier 1.5 Flow

A new `NativeJarExtractor` class handles classpath extraction:

1. Check if `~/.casehub/native/sherpa-onnx/<version>/<platform>/` already exists.
   If so, skip — Tier 2 will find the libs.
2. Scan classpath for `META-INF/native/sherpa-onnx/<version>/<platform>/<lib-filename>`
   using `Thread.currentThread().getContextClassLoader().getResources()`.
3. If found: extract both libs to `Provisioner.nativeCacheDir()` (the existing
   `~/.casehub/native/sherpa-onnx/<version>/<platform>/` directory).
4. Use `Provisioner`'s existing file-locking pattern for concurrent JVM safety.

### Cache Behaviour

- **First run:** classpath JAR → extract to cache → load (one-time ~100ms cost)
- **Subsequent runs:** Tier 2 finds extracted libs directly (no JAR scanning)
- **Override:** system install (Tier 1) or pre-cached libs (Tier 2) take priority
  over classpath extraction

### Integration Point

`SherpaLibrary.load()` gains a new block between Tier 1 and Tier 2:

```java
// Tier 1: system library path (existing)
// ...

// Tier 1.5: classpath JAR extraction
Path cacheDir = SherpaLibrary.defaultCacheDir();
if (!Files.isDirectory(cacheDir)) {
    NativeJarExtractor.extractIfAvailable(cacheDir);
}
if (Files.isDirectory(cacheDir)) {
    // load from cacheDir (same as existing Tier 2 code)
}

// Tier 2: local cache (existing)
// Tier 3: auto-download (existing)
```

## Module Build Process

### Build-Time Provisioning

Native libs are NOT checked into git (31MB binaries). Instead, each module
uses `exec-maven-plugin` during `generate-resources` to invoke a provisioning
main class that:

1. Calls `Provisioner.ensureNativeLibrary()` to download if not cached
2. Copies the native libs to `target/classes/META-INF/native/sherpa-onnx/<version>/<platform>/`

### POM Structure

Each native module's `pom.xml` is mechanical — identical structure with only
the platform ID varying:

```xml
<artifactId>speech-sherpa-native-osx-arm64</artifactId>
<packaging>jar</packaging>

<build>
  <plugins>
    <plugin>
      <groupId>org.codehaus.mojo</groupId>
      <artifactId>exec-maven-plugin</artifactId>
      <executions>
        <execution>
          <id>provision-native</id>
          <phase>generate-resources</phase>
          <goals><goal>java</goal></goals>
          <configuration>
            <mainClass>io.casehub.blocks.speech.sherpa.NativePackager</mainClass>
            <arguments>
              <argument>osx-arm64</argument>
            </arguments>
          </configuration>
        </execution>
      </executions>
    </plugin>
  </plugins>
</build>
```

### CI Considerations

- macOS runners build `osx-arm64` and `osx-x64` modules
- Linux runners build `linux-x64` and `linux-arm64` modules
- Windows deferred until needed
- Each module only builds on matching platform (or cross-downloads)

### Publishing

Standard `mvn deploy` to GitHub Packages. Platform is in the artifactId
(not a classifier), so no special publishing configuration needed.

## Components

### NativeJarExtractor (new class in speech-sherpa)

Responsibilities:
- Scan classpath for native lib resources at the conventional path
- Extract to Provisioner cache directory with file locking
- Skip if cache directory already exists

### NativePackager (new main class in speech-sherpa)

Responsibilities:
- Build-time entry point for `exec-maven-plugin`
- Takes platform ID as argument
- Calls `Provisioner.ensureNativeLibrary()` and copies libs to target

### SherpaLibrary (modified)

Changes:
- Add Tier 1.5 classpath extraction check between Tier 1 and Tier 2
- Delegate to `NativeJarExtractor.extractIfAvailable()`

## Testing

### Unit Tests

- `NativeJarExtractorTest`: Verify classpath scanning finds resources at
  expected path, extracts to target dir, skips if already extracted. Uses
  a test JAR with a dummy file at
  `META-INF/native/sherpa-onnx/<version>/<platform>/dummy.so`.

### Integration Tests (guarded by hasModels)

- Full Tier 1.5 flow: classpath resource → extract → SymbolLookup → load
- Idempotency: second extraction is a no-op (cache hit)

### Build Verification

- Each native module's JAR contains expected files at the right resource path
  (verified via `jar tf` in CI)

## Error Handling

- Classpath scan finds no native resources: silent fallthrough to Tier 2/3
- Extraction fails (permissions, disk full): log warning, fallthrough to Tier 2/3
- Extracted libs are corrupt: `SymbolLookup.libraryLookup` throws, caught by
  existing error handling in `SherpaLibrary.load()`

## Scope

### In scope
- 5 native module POMs
- `NativeJarExtractor` classpath extraction
- `NativePackager` build-time entry point
- `SherpaLibrary` Tier 1.5 integration
- Unit and integration tests

### Out of scope
- OS-detection auto-dependency (e.g., Maven profiles that auto-select platform) — consumer picks explicitly
- Uber-JAR with all platforms bundled — not needed for CaseHub's deployment model
- Model bundling in JARs — models are larger (~75MB+) and handled by Provisioner

## References

- [SherpaLibrary.java] — existing 3-tier loading
- [Provisioner.java] — download, extract, cache, file locking
- [GE-20260831-f91b16] — JarURLConnection gotcha: use JarFile directly for classpath extraction
- [casehubio/blocks#174] — parent epic: Zero-Install Speech Experience
- [casehubio/blocks#176] — this issue
- Netty native transports — pattern reference for separate platform artifactIds
- SQLite JDBC — pattern reference for classpath extraction to temp/cache
