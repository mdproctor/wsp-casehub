## D1: Maven module structure for native JARs

**Choice:** Separate native modules per platform
**Alternatives:**
- Single module with classifier profiles — can't build all classifiers in one invocation, CI complexity
- Assembly-based packaging — non-standard classifier artifacts, poor IDE support
**Rationale:** Clean separation, consumer adds one dependency, each JAR independently versioned. Matches Netty native transports pattern.
**Trade-offs:** 5 new Maven modules (mechanical, identical structure)
**Sources:** casehubio/blocks#176, Netty native transports pattern, SherpaLibrary.java loading tiers
**Exploration:** quick
**Status:** captured

## D2: Native lib bundling scope

**Choice:** Bundle sherpa-onnx + onnxruntime together in each platform JAR
**Alternatives:**
- Separate JARs for sherpa-onnx and onnxruntime — more flexible but version-locked pair anyway
**Rationale:** These libs are tightly coupled (sherpa-onnx links against a specific onnxruntime version). One dependency per platform is simpler for consumers.
**Trade-offs:** Can't upgrade onnxruntime independently (not a real concern — sherpa-onnx pins the version)
**Sources:** Provisioner.java NATIVE_ASSETS map, SherpaLibrary loading
**Exploration:** quick
**Status:** captured

## D3: SherpaLibrary loading tier order

**Choice:** Classpath JAR extraction as Tier 1.5 (after system path, before local cache)
**Alternatives:**
- First (before system path) — can't override with system install
- After local cache (before auto-download) — unusual ordering
**Rationale:** Classpath JARs are version-managed by Maven (reliable), but system installs should still win for development/debugging.
**Trade-offs:** System install takes priority even if version mismatches — acceptable since system install is deliberate
**Sources:** SherpaLibrary.java tiers 1-3
**Exploration:** quick
**Status:** captured

## D4: Runtime extraction cache strategy

**Choice:** Extract to existing Provisioner cache dir (~/.casehub/native/sherpa-onnx/<version>/<platform>/)
**Alternatives:**
- Extract to java.io.tmpdir — simpler but re-extracts every JVM restart
- Load directly from JAR without extracting — SymbolLookup may not support jar: URLs
**Rationale:** Once extracted, Tier 2 (local cache) finds them on subsequent runs. Single extraction cost. Reuses existing cache structure.
**Trade-offs:** Writes to user home directory (standard for native lib caching)
**Sources:** Provisioner.nativeCacheDir(), SherpaLibrary.defaultCacheDir()
**Exploration:** quick
**Status:** captured
