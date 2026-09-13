# Platform-Specific Maven JARs Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** casehubio/blocks#176 — Platform-specific Maven JARs for native lib bundling
**Issue group:** casehubio/blocks#176

**Goal:** Publish platform-specific Maven JAR artifacts containing sherpa-onnx
and onnxruntime native libs, with runtime classpath extraction in SherpaLibrary.

**Architecture:** 5 new Maven modules (one per platform) package native libs at
`META-INF/native/sherpa-onnx/<version>/<platform>/`. A new `NativeJarExtractor`
scans the classpath at runtime and extracts to the existing Provisioner cache dir.
`SherpaLibrary.load()` gains a Tier 1.5 between system path and local cache.

**Tech Stack:** Java 22 (FFM/Panama), Maven, `exec-maven-plugin`

## Global Constraints

- JDK 22+ required (speech-sherpa uses FFM APIs)
- groupId: `io.casehub`, version: `${project.version}` (0.2-SNAPSHOT)
- Parent POM: `casehub-blocks-parent`
- speech-sherpa modules gated behind `jdk22+` profile
- Native lib version: `SherpaLibrary.VERSION` (1.13.6)
- Platform IDs: `osx-arm64`, `osx-x64`, `linux-x64`, `linux-arm64`, `win-x64`
- Resource path convention: `META-INF/native/sherpa-onnx/<version>/<platform>/`
- IntelliJ MCP: use `mcp__intellij-index__*` for all code navigation and editing
- Project path for IntelliJ: `/Users/mdproctor/claude/casehub/slots/167/blocks`

---

## Batch 1: Runtime Classpath Extraction

### Task 1: NativeJarExtractor — classpath scan and extract

**Files:**
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/NativeJarExtractor.java`
- Test: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/NativeJarExtractorTest.java`
- Create: `speech-sherpa/src/test/resources/META-INF/native/sherpa-onnx/1.13.6/test-platform/libdummy.so` (test fixture)

**Interfaces:**
- Consumes: `SherpaLibrary.VERSION`, `SherpaLibrary.platformId()`, `SherpaLibrary.sherpaLibName()`, `SherpaLibrary.onnxRuntimeLibName()` (currently private — will need package-private access)
- Produces: `NativeJarExtractor.extractIfAvailable(Path targetDir): boolean` — called by SherpaLibrary in Task 2

- [ ] **Step 1: Make helper methods package-private in SherpaLibrary**

Change `sherpaLibName()` and `onnxRuntimeLibName()` from `private static` to
package-private `static` so `NativeJarExtractor` can access them.

Use `ide_edit_member` to replace each method signature:

`SherpaLibrary.java:263` — change `private static String sherpaLibName()` to `static String sherpaLibName()`
`SherpaLibrary.java:270` — change `private static String onnxRuntimeLibName()` to `static String onnxRuntimeLibName()`

- [ ] **Step 2: Create test fixture**

Create a dummy file at the expected classpath resource path for testing:

```bash
mkdir -p /Users/mdproctor/claude/casehub/slots/167/blocks/speech-sherpa/src/test/resources/META-INF/native/sherpa-onnx/1.13.6/test-platform
echo "dummy" > /Users/mdproctor/claude/casehub/slots/167/blocks/speech-sherpa/src/test/resources/META-INF/native/sherpa-onnx/1.13.6/test-platform/libdummy.so
```

- [ ] **Step 3: Write failing test for NativeJarExtractor**

Create `NativeJarExtractorTest.java`:

```java
package io.casehub.blocks.speech.sherpa;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;

class NativeJarExtractorTest {

    @TempDir Path tempDir;

    @Test
    void extractsResourceFromClasspath() {
        Path targetDir = tempDir.resolve("extracted");

        boolean extracted = NativeJarExtractor.extractResource(
                "META-INF/native/sherpa-onnx/1.13.6/test-platform/libdummy.so",
                targetDir.resolve("libdummy.so"));

        assertThat(extracted).isTrue();
        assertThat(targetDir.resolve("libdummy.so")).exists();
        assertThat(Files.readString(targetDir.resolve("libdummy.so"))).contains("dummy");
    }

    @Test
    void returnsFalseWhenResourceNotFound() {
        Path targetDir = tempDir.resolve("extracted");

        boolean extracted = NativeJarExtractor.extractResource(
                "META-INF/native/sherpa-onnx/1.13.6/nonexistent/libfoo.so",
                targetDir.resolve("libfoo.so"));

        assertThat(extracted).isFalse();
        assertThat(targetDir.resolve("libfoo.so")).doesNotExist();
    }

    @Test
    void skipsExtractionWhenTargetDirExists() {
        Path targetDir = tempDir.resolve("already-exists");
        targetDir.toFile().mkdirs();

        boolean extracted = NativeJarExtractor.extractIfAvailable(targetDir);

        assertThat(extracted).isFalse();
    }
}
```

- [ ] **Step 4: Run test to verify it fails**

Run: `mvn -f speech-sherpa/pom.xml test -Dtest=NativeJarExtractorTest -Dsurefire.useFile=false -q`
Expected: FAIL — `NativeJarExtractor` class not found

- [ ] **Step 5: Implement NativeJarExtractor**

Create `NativeJarExtractor.java`:

```java
package io.casehub.blocks.speech.sherpa;

import java.io.IOException;
import java.io.InputStream;
import java.io.UncheckedIOException;
import java.net.URL;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;

final class NativeJarExtractor {

    private static final System.Logger LOG = System.getLogger("casehub-speech");

    private NativeJarExtractor() {}

    static boolean extractIfAvailable(Path targetDir) {
        if (Files.isDirectory(targetDir)) {
            return false;
        }

        String version = SherpaLibrary.VERSION;
        String platform = SherpaLibrary.platformId();
        String prefix = "META-INF/native/sherpa-onnx/" + version + "/" + platform + "/";

        String sherpaLib = SherpaLibrary.sherpaLibName();
        String onnxLib = SherpaLibrary.onnxRuntimeLibName();

        URL sherpaUrl = Thread.currentThread().getContextClassLoader()
                .getResource(prefix + sherpaLib);
        URL onnxUrl = Thread.currentThread().getContextClassLoader()
                .getResource(prefix + onnxLib);

        if (sherpaUrl == null || onnxUrl == null) {
            return false;
        }

        try {
            Path tempDir = Files.createTempDirectory(targetDir.getParent(), ".native-extract-");
            try {
                extractResource(prefix + sherpaLib, tempDir.resolve(sherpaLib));
                extractResource(prefix + onnxLib, tempDir.resolve(onnxLib));
                Files.move(tempDir, targetDir, StandardCopyOption.ATOMIC_MOVE);
                LOG.log(System.Logger.Level.INFO,
                        "Extracted native libs from classpath to {0}", targetDir);
                return true;
            } catch (Exception e) {
                deleteRecursively(tempDir);
                throw e;
            }
        } catch (IOException e) {
            LOG.log(System.Logger.Level.WARNING,
                    "Failed to extract native libs from classpath: {0}", e.getMessage());
            return false;
        }
    }

    static boolean extractResource(String resourcePath, Path target) {
        URL url = Thread.currentThread().getContextClassLoader()
                .getResource(resourcePath);
        if (url == null) {
            return false;
        }

        try {
            Files.createDirectories(target.getParent());
            try (InputStream in = url.openStream()) {
                Files.copy(in, target, StandardCopyOption.REPLACE_EXISTING);
            }
            return true;
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to extract " + resourcePath, e);
        }
    }

    private static void deleteRecursively(Path dir) {
        try (var walk = Files.walk(dir)) {
            walk.sorted(java.util.Comparator.reverseOrder())
                .forEach(p -> { try { Files.deleteIfExists(p); } catch (IOException ignored) {} });
        } catch (IOException ignored) {}
    }
}
```

- [ ] **Step 6: Run test to verify it passes**

Run: `mvn -f speech-sherpa/pom.xml test -Dtest=NativeJarExtractorTest -Dsurefire.useFile=false -q`
Expected: PASS (3 tests)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add \
  speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/NativeJarExtractor.java \
  speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java \
  speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/NativeJarExtractorTest.java \
  speech-sherpa/src/test/resources/META-INF/
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#176): add NativeJarExtractor for classpath native lib extraction

Refs #176"
```

---

### Task 2: SherpaLibrary Tier 1.5 integration

**Files:**
- Modify: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java:173-217` (load method)
- Modify: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechToTextTest.java` (integration test)

**Interfaces:**
- Consumes: `NativeJarExtractor.extractIfAvailable(Path)` from Task 1
- Produces: Updated `SherpaLibrary.load()` with Tier 1.5 classpath extraction

- [ ] **Step 1: Write failing test for Tier 1.5 flow**

Add a unit test to verify the loading tier attempts classpath extraction.
Since the native lib may not be on the classpath in test, test the negative
path — verify it falls through gracefully:

Add to `SherpaOnnxSpeechToTextTest.java`:

```java
@Test
void classpathExtractionFallsThroughWhenNoNativeJar(@TempDir Path cacheDir) {
    boolean extracted = NativeJarExtractor.extractIfAvailable(cacheDir.resolve("nonexistent-platform"));
    assertThat(extracted).isFalse();
}
```

- [ ] **Step 2: Run test to verify it passes (negative path)**

Run: `mvn -f speech-sherpa/pom.xml test -Dtest=SherpaOnnxSpeechToTextTest#classpathExtractionFallsThroughWhenNoNativeJar -Dsurefire.useFile=false -q`
Expected: PASS

- [ ] **Step 3: Add Tier 1.5 to SherpaLibrary.load()**

Insert the classpath extraction block between Tier 1 (system library path)
and Tier 2 (local cache) in the `load()` method. Use `ide_edit_member` to
replace the `load()` method:

```java
static SherpaLibrary load() {
    if (INSTANCE != null) {return INSTANCE;}
    synchronized (SherpaLibrary.class) {
        if (INSTANCE != null) {return INSTANCE;}

        // Tier 1: system library path
        try {
            SymbolLookup lookup = SymbolLookup.libraryLookup("sherpa-onnx-c-api", Arena.global());
            INSTANCE = new SherpaLibrary(lookup);
            return INSTANCE;
        } catch (IllegalArgumentException | UnsatisfiedLinkError ignored) {
        }

        // Tier 1.5: classpath JAR extraction
        Path cacheDir = defaultCacheDir();
        if (!java.nio.file.Files.isDirectory(cacheDir)) {
            java.nio.file.Files.createDirectories(cacheDir.getParent());
            NativeJarExtractor.extractIfAvailable(cacheDir);
        }

        // Tier 2: local cache
        if (cacheDir != null && java.nio.file.Files.isDirectory(cacheDir)) {
            Path onnxRuntime = cacheDir.resolve(onnxRuntimeLibName());
            Path sherpaLib   = cacheDir.resolve(sherpaLibName());
            if (java.nio.file.Files.exists(sherpaLib) && java.nio.file.Files.exists(onnxRuntime)) {
                SymbolLookup.libraryLookup(onnxRuntime, Arena.global());
                SymbolLookup lookup = SymbolLookup.libraryLookup(sherpaLib, Arena.global());
                INSTANCE = new SherpaLibrary(lookup);
                return INSTANCE;
            }
        }

        // Tier 3: auto-download (opt-in via system property)
        if (Provisioner.isAutoDownloadEnabled()) {
            Path downloadedDir = Provisioner.ensureNativeLibrary();
            Path onnxRuntime = downloadedDir.resolve(onnxRuntimeLibName());
            Path sherpaLib   = downloadedDir.resolve(sherpaLibName());
            if (java.nio.file.Files.exists(sherpaLib) && java.nio.file.Files.exists(onnxRuntime)) {
                SymbolLookup.libraryLookup(onnxRuntime, Arena.global());
                SymbolLookup lookup = SymbolLookup.libraryLookup(sherpaLib, Arena.global());
                INSTANCE = new SherpaLibrary(lookup);
                return INSTANCE;
            }
        }

        throw new UnsatisfiedLinkError(
                "sherpa-onnx native library not found. Install it system-wide or place "
                + sherpaLibName() + " + " + onnxRuntimeLibName()
                + " in " + defaultCacheDir());
    }
}
```

Note: The existing Tier 2 code used `resolveNativeDir()` which returns
`defaultCacheDir()` if the directory exists. After Tier 1.5, we've already
computed `cacheDir = defaultCacheDir()` and potentially extracted to it,
so we use `cacheDir` directly to avoid redundant directory resolution.

- [ ] **Step 4: Run all SherpaLibrary-related tests**

Run: `mvn -f speech-sherpa/pom.xml test -Dtest="SherpaOnnxSpeechToTextTest,NativeJarExtractorTest" -Dsurefire.useFile=false -q`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add \
  speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java \
  speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/SherpaOnnxSpeechToTextTest.java
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#176): add Tier 1.5 classpath extraction to SherpaLibrary.load()

Refs #176"
```

---

## Batch 2: Maven Module Packaging

### Task 3: NativePackager and reference module (osx-arm64)

**Files:**
- Create: `speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/NativePackager.java`
- Create: `speech-sherpa-native-osx-arm64/pom.xml`
- Modify: `pom.xml` (parent — add module to jdk22+ profile)
- Test: `speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/NativePackagerTest.java`

**Interfaces:**
- Consumes: `Provisioner.ensureNativeLibrary()`, `Provisioner.nativeCacheDir()`, `SherpaLibrary.platformId()`, `SherpaLibrary.sherpaLibName()`, `SherpaLibrary.onnxRuntimeLibName()`
- Produces: `NativePackager.main(String[] args)` — build-time entry point invoked by exec-maven-plugin

- [ ] **Step 1: Write failing test for NativePackager**

Create `NativePackagerTest.java`:

```java
package io.casehub.blocks.speech.sherpa;

import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.io.TempDir;

import java.nio.file.Files;
import java.nio.file.Path;

import static org.assertj.core.api.Assertions.assertThat;
import static org.assertj.core.api.Assertions.assertThatThrownBy;

class NativePackagerTest {

    @TempDir Path tempDir;

    @Test
    void rejectsNoArguments() {
        assertThatThrownBy(() -> NativePackager.packageNative(new String[]{}, tempDir))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("platform ID");
    }

    @Test
    void rejectsUnknownPlatform() {
        assertThatThrownBy(() -> NativePackager.packageNative(new String[]{"unknown-arch"}, tempDir))
                .isInstanceOf(IllegalArgumentException.class)
                .hasMessageContaining("unknown-arch");
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -f speech-sherpa/pom.xml test -Dtest=NativePackagerTest -Dsurefire.useFile=false -q`
Expected: FAIL — `NativePackager` class not found

- [ ] **Step 3: Implement NativePackager**

Create `NativePackager.java`:

```java
package io.casehub.blocks.speech.sherpa;

import java.io.IOException;
import java.io.UncheckedIOException;
import java.nio.file.Files;
import java.nio.file.Path;
import java.nio.file.StandardCopyOption;
import java.util.Set;

public final class NativePackager {

    private static final Set<String> VALID_PLATFORMS = Set.of(
            "osx-arm64", "osx-x64", "linux-x64", "linux-arm64", "win-x64");

    private NativePackager() {}

    public static void main(String[] args) {
        Path outputDir = Path.of(System.getProperty("project.build.outputDirectory",
                "target/classes"));
        packageNative(args, outputDir);
    }

    static void packageNative(String[] args, Path outputDir) {
        if (args.length < 1) {
            throw new IllegalArgumentException(
                    "Usage: NativePackager <platform ID>. "
                    + "Valid platforms: " + VALID_PLATFORMS);
        }

        String platformId = args[0];
        if (!VALID_PLATFORMS.contains(platformId)) {
            throw new IllegalArgumentException(
                    "Unknown platform: " + platformId
                    + ". Valid platforms: " + VALID_PLATFORMS);
        }

        Path nativeDir = Provisioner.nativeCacheDir(platformId);
        if (!Files.isDirectory(nativeDir)) {
            if (Provisioner.isAutoDownloadEnabled()) {
                Provisioner.ensureNativeLibrary();
                nativeDir = Provisioner.nativeCacheDir(platformId);
            }
            if (!Files.isDirectory(nativeDir)) {
                throw new IllegalStateException(
                        "Native libs not found at " + nativeDir
                        + ". Run with -Dcasehub.speech.auto-download=true or "
                        + "provision manually.");
            }
        }

        Path targetDir = outputDir.resolve("META-INF/native/sherpa-onnx")
                .resolve(SherpaLibrary.VERSION).resolve(platformId);

        try {
            Files.createDirectories(targetDir);
            copyPlatformLibs(nativeDir, targetDir, platformId);
        } catch (IOException e) {
            throw new UncheckedIOException("Failed to package native libs", e);
        }
    }

    private static void copyPlatformLibs(Path sourceDir, Path targetDir,
                                          String platformId) throws IOException {
        String sherpaLib = sherpaLibNameFor(platformId);
        String onnxLib = onnxRuntimeLibNameFor(platformId);

        Path sherpaSrc = sourceDir.resolve(sherpaLib);
        Path onnxSrc = sourceDir.resolve(onnxLib);

        if (!Files.exists(sherpaSrc)) {
            throw new IllegalStateException("Missing " + sherpaLib + " in " + sourceDir);
        }
        if (!Files.exists(onnxSrc)) {
            throw new IllegalStateException("Missing " + onnxLib + " in " + sourceDir);
        }

        Files.copy(sherpaSrc, targetDir.resolve(sherpaLib), StandardCopyOption.REPLACE_EXISTING);
        Files.copy(onnxSrc, targetDir.resolve(onnxLib), StandardCopyOption.REPLACE_EXISTING);
    }

    static String sherpaLibNameFor(String platformId) {
        if (platformId.startsWith("osx")) return "libsherpa-onnx-c-api.dylib";
        if (platformId.startsWith("win")) return "sherpa-onnx-c-api.dll";
        return "libsherpa-onnx-c-api.so";
    }

    static String onnxRuntimeLibNameFor(String platformId) {
        if (platformId.startsWith("osx")) return "libonnxruntime.dylib";
        if (platformId.startsWith("win")) return "onnxruntime.dll";
        return "libonnxruntime.so";
    }
}
```

Note: `NativePackager` uses its own `sherpaLibNameFor(platformId)` and
`onnxRuntimeLibNameFor(platformId)` instead of `SherpaLibrary.sherpaLibName()`
because the packager needs to produce libs for ANY platform, not just
the current one.

- [ ] **Step 4: Add Provisioner.nativeCacheDir(String) overload**

The existing `Provisioner.nativeCacheDir()` uses `SherpaLibrary.platformId()`
(current platform). NativePackager needs to resolve for a specific platform.
Add a static overload:

Use `ide_insert_member` to add after the existing `nativeCacheDir()` method:

```java
static Path nativeCacheDir(String platformId) {
    return cacheBaseDir().resolve("native").resolve("sherpa-onnx")
        .resolve(SherpaLibrary.VERSION).resolve(platformId);
}
```

- [ ] **Step 5: Run test to verify it passes**

Run: `mvn -f speech-sherpa/pom.xml test -Dtest=NativePackagerTest -Dsurefire.useFile=false -q`
Expected: PASS (2 tests)

- [ ] **Step 6: Create osx-arm64 native module POM**

Create `speech-sherpa-native-osx-arm64/pom.xml`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <parent>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-blocks-parent</artifactId>
    <version>0.2-SNAPSHOT</version>
  </parent>

  <artifactId>speech-sherpa-native-osx-arm64</artifactId>
  <name>speech-sherpa-native-osx-arm64</name>
  <description>sherpa-onnx + onnxruntime native libs for macOS ARM64</description>

  <properties>
    <maven.compiler.release>22</maven.compiler.release>
    <native.platform>osx-arm64</native.platform>
  </properties>

  <dependencies>
    <dependency>
      <groupId>io.casehub</groupId>
      <artifactId>casehub-blocks-speech-sherpa</artifactId>
      <version>${project.version}</version>
      <scope>provided</scope>
    </dependency>
  </dependencies>

  <build>
    <plugins>
      <plugin>
        <groupId>org.codehaus.mojo</groupId>
        <artifactId>exec-maven-plugin</artifactId>
        <version>3.5.0</version>
        <executions>
          <execution>
            <id>provision-native</id>
            <phase>generate-resources</phase>
            <goals><goal>java</goal></goals>
            <configuration>
              <mainClass>io.casehub.blocks.speech.sherpa.NativePackager</mainClass>
              <arguments>
                <argument>${native.platform}</argument>
              </arguments>
              <systemProperties>
                <systemProperty>
                  <key>project.build.outputDirectory</key>
                  <value>${project.build.outputDirectory}</value>
                </systemProperty>
                <systemProperty>
                  <key>casehub.speech.auto-download</key>
                  <value>true</value>
                </systemProperty>
              </systemProperties>
              <classpathScope>provided</classpathScope>
            </configuration>
          </execution>
        </executions>
      </plugin>
    </plugins>
  </build>
</project>
```

- [ ] **Step 7: Add module to parent POM jdk22+ profile**

In `pom.xml` (blocks root), add `speech-sherpa-native-osx-arm64` to the
`jdk22+` profile's `<modules>` block, after `speech-demo`:

Use `ide_replace_text_in_file`:
- Search: `<module>speech-demo</module>`
- Replace: `<module>speech-demo</module>\n        <module>speech-sherpa-native-osx-arm64</module>`

- [ ] **Step 8: Verify the native module builds**

Run: `mvn -f speech-sherpa-native-osx-arm64/pom.xml package -q`

Verify the JAR contains the native libs:
```bash
jar tf speech-sherpa-native-osx-arm64/target/speech-sherpa-native-osx-arm64-0.2-SNAPSHOT.jar | grep META-INF/native
```
Expected output:
```
META-INF/native/sherpa-onnx/1.13.6/osx-arm64/libsherpa-onnx-c-api.dylib
META-INF/native/sherpa-onnx/1.13.6/osx-arm64/libonnxruntime.dylib
```

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add \
  speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/NativePackager.java \
  speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java \
  speech-sherpa/src/test/java/io/casehub/blocks/speech/sherpa/NativePackagerTest.java \
  speech-sherpa-native-osx-arm64/pom.xml \
  pom.xml
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#176): add NativePackager and osx-arm64 native module

Refs #176"
```

---

### Task 4: Remaining 4 platform modules

**Files:**
- Create: `speech-sherpa-native-osx-x64/pom.xml`
- Create: `speech-sherpa-native-linux-x64/pom.xml`
- Create: `speech-sherpa-native-linux-arm64/pom.xml`
- Create: `speech-sherpa-native-win-x64/pom.xml`
- Modify: `pom.xml` (parent — add all 4 modules to jdk22+ profile)

**Interfaces:**
- Consumes: Same POM structure as `speech-sherpa-native-osx-arm64` from Task 3
- Produces: 4 additional native module JARs

- [ ] **Step 1: Create osx-x64 module POM**

Create `speech-sherpa-native-osx-x64/pom.xml` — identical to osx-arm64 with:
- `<artifactId>speech-sherpa-native-osx-x64</artifactId>`
- `<name>speech-sherpa-native-osx-x64</name>`
- `<description>sherpa-onnx + onnxruntime native libs for macOS x86_64</description>`
- `<native.platform>osx-x64</native.platform>`

- [ ] **Step 2: Create linux-x64 module POM**

Create `speech-sherpa-native-linux-x64/pom.xml` — identical structure with:
- `<artifactId>speech-sherpa-native-linux-x64</artifactId>`
- `<name>speech-sherpa-native-linux-x64</name>`
- `<description>sherpa-onnx + onnxruntime native libs for Linux x86_64</description>`
- `<native.platform>linux-x64</native.platform>`

- [ ] **Step 3: Create linux-arm64 module POM**

Create `speech-sherpa-native-linux-arm64/pom.xml` — identical structure with:
- `<artifactId>speech-sherpa-native-linux-arm64</artifactId>`
- `<name>speech-sherpa-native-linux-arm64</name>`
- `<description>sherpa-onnx + onnxruntime native libs for Linux ARM64</description>`
- `<native.platform>linux-arm64</native.platform>`

- [ ] **Step 4: Create win-x64 module POM**

Create `speech-sherpa-native-win-x64/pom.xml` — identical structure with:
- `<artifactId>speech-sherpa-native-win-x64</artifactId>`
- `<name>speech-sherpa-native-win-x64</name>`
- `<description>sherpa-onnx + onnxruntime native libs for Windows x86_64</description>`
- `<native.platform>win-x64</native.platform>`

- [ ] **Step 5: Add all 4 modules to parent POM**

In `pom.xml` (blocks root), add all 4 modules to the `jdk22+` profile
after `speech-sherpa-native-osx-arm64`:

Use `ide_replace_text_in_file`:
- Search: `<module>speech-sherpa-native-osx-arm64</module>`
- Replace with all 5 native modules (including the existing osx-arm64):

```xml
<module>speech-sherpa-native-osx-arm64</module>
        <module>speech-sherpa-native-osx-x64</module>
        <module>speech-sherpa-native-linux-x64</module>
        <module>speech-sherpa-native-linux-arm64</module>
        <module>speech-sherpa-native-win-x64</module>
```

- [ ] **Step 6: Verify POM resolution**

Run: `mvn -f pom.xml validate -q`
Expected: no errors (all modules resolve)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/slots/167/blocks add \
  speech-sherpa-native-osx-x64/pom.xml \
  speech-sherpa-native-linux-x64/pom.xml \
  speech-sherpa-native-linux-arm64/pom.xml \
  speech-sherpa-native-win-x64/pom.xml \
  pom.xml
git -C /Users/mdproctor/claude/casehub/slots/167/blocks commit -m "feat(#176): add remaining 4 platform native modules

Refs #176"
```

---

## References

- [specs/issue-176-native-jars/2026-09-03-native-jars-design.md] — design spec
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/SherpaLibrary.java:173-217] — existing load() tiers
- [speech-sherpa/src/main/java/io/casehub/blocks/speech/sherpa/Provisioner.java:136-139] — nativeCacheDir()
- [GE-20260831-f91b16] — JarURLConnection fails on JARs without directory entries
- [casehubio/blocks#176] — focal issue
- [casehubio/blocks#174] — parent epic (Zero-Install Speech Experience)
