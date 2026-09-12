---
layout: post
title: "The Native Lib Problem Is a Distribution Problem"
date: 2026-09-03
entry_type: article
subtype: diary
projects: [casehubio/blocks-ui, casehubio/blocks]
tags: [speech, sherpa-onnx, maven, native-libs, ffm, distribution, zero-install]
series: issue-190-speech-denoising
---

# The Native Lib Problem Is a Distribution Problem

sherpa-onnx is a single C library with a single job: run speech models on-device. Getting it onto a developer's machine is three manual steps — download the right archive for your platform, extract it, set the library path. That's not hard, but it's the step where adoption dies. Nobody reads setup instructions. They add a Maven dependency, call the API, and expect it to work.

## The loading tiers

SherpaLibrary already had a tiered loading strategy before this work:

1. **System path** — if `libsherpa-onnx-c-api` is installed system-wide, `SymbolLookup.libraryLookup` finds it by name. Zero configuration. The developer's own install wins.
2. **Local cache** — `~/.casehub/native/sherpa-onnx/1.13.6/osx-arm64/`. Provisioner puts files here after downloading.
3. **Auto-download** — opt-in. Provisioner fetches the archive from GitHub Releases, verifies the SHA-256 checksum, extracts to the local cache.

Tier 3 already solves the zero-install problem at runtime — if you set a system property. But it means every first run downloads 31MB from GitHub, and it fails behind corporate proxies. What we needed was Tier 1.5: the native libs already on the classpath, bundled in a JAR.

## The Netty pattern

Netty solved this years ago. Platform-specific classifier JARs — `netty-transport-native-epoll:linux-x86_64` — one dependency per platform, zero manual install. The consumer's POM declares the platform; Maven resolves the artifact; the native libs arrive on the classpath like any other dependency.

We followed the same structure: five Maven modules, one per platform. Each module's JAR contains the native libs at `META-INF/native/sherpa-onnx/1.13.6/<platform>/`. The version and platform are in the resource path, so multiple versions can coexist on the classpath without collisions.

The consumer adds one `runtime` dependency:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>speech-sherpa-native-osx-arm64</artifactId>
    <scope>runtime</scope>
</dependency>
```

## The cache reuse trick

The interesting design decision is what happens at runtime when `SherpaLibrary.load()` finds native libs on the classpath. The obvious approach — extract to a temp directory, load, done — means re-extraction on every JVM restart. SQLite JDBC does this. It works, but 31MB of I/O on every startup is noticeable.

Instead, Tier 1.5 extracts to the same directory Provisioner uses: `~/.casehub/native/sherpa-onnx/1.13.6/osx-arm64/`. Once extracted, Tier 2 finds the libs on subsequent runs without scanning the classpath at all. First-run cost, no ongoing cost. The extraction uses Provisioner's file-locking pattern for concurrent JVM safety — two processes starting simultaneously don't corrupt each other's extraction.

The tier ordering is deliberate: system path wins over classpath JARs, which win over local cache, which win over auto-download. A developer who installs the library system-wide for debugging gets their install, not the JAR's bundled version.

## The recognizer was the other bottleneck

Separately from the distribution problem, `SherpaOnnxSpeechToText.transcribe()` was creating a new offline recognizer per call — allocating a config struct, loading the model, running inference, destroying the recognizer. Model loading alone is ~500ms. For batch transcription of multiple files, that overhead dominates.

The fix is a single cached recognizer keyed by `(modelSize, languageHint)` — the two `TranscriptionOptions` fields that affect the native config. Same options? Reuse. Different options? Destroy the old one, create a new one. The class implements `AutoCloseable` for cleanup.

The cache needs its own `Arena.ofShared()` for the recognizer's config memory — the per-call confined arena closes after each transcription, and the recognizer must survive across calls. Whether sherpa-onnx copies the config strings internally or holds pointers into the arena is undocumented, so the arena stays alive as long as the recognizer does.

## The build-time trick

31MB of native libs can't live in git. Each platform module uses `exec-maven-plugin` during `generate-resources` to invoke a `NativePackager` class that calls Provisioner to download the libs (if not already cached) and copies them into `target/classes/META-INF/native/`. The JAR is built from the result. CI builds each module on its matching platform runner.

The consumer never sees this machinery. They add a dependency, Maven resolves it, the native libs land on the classpath, SherpaLibrary extracts them once, and every subsequent run loads from cache. That's the zero-install experience the epic was aiming for.
