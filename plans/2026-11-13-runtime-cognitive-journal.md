# Runtime Cognitive Journal Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #961 — Runtime cognitive pipeline journal — tick-correlated observability for full trace
**Issue group:** #961

**Goal:** Replace five competing observability mechanisms with a single unified cognitive journal — typed entries, per-scope accumulation, and persistent storage — enabling causal tracing through the appraisal-regulation-mood-drive pipeline.

**Architecture:** Single `CognitiveJournal` interface with one `record(CognitiveEntry)` method. 16 sealed record types across derivation, runtime, and consolidation domains. `ScopedCognitiveJournal` routes entries to ThreadLocal scope-local `CognitiveTrace` accumulators and persistent `CognitiveJournalStore`. DerivationObserver bridged (not replaced) via `DerivationJournalBridge`.

**Tech Stack:** Java 26, Quarkus CDI, Jackson for JSON serialization, Maven multi-module

## Global Constraints

- All new types in `cognition-api` (L25) package `io.casehub.neocortex.cognition.journal`
- ScopedCognitiveJournal in `cognition` (L25) package `io.casehub.neocortex.cognition.core`
- CognitiveJournalStore in `cognitive-observability-core` (L23) package `io.casehub.neocortex.cognitive.observability`
- Every CognitiveEntry carries `tenantId`, `agentId`, `EventRef eventRef`, `Instant timestamp`
- Use `ide_insert_member` / `ide_replace_member` for structural edits, `ide_refactor_rename` for renames
- Build with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/pom.xml`
- Test with: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f neocortex/<module>/pom.xml test -Dtest=<TestClass>`

**Base paths (all relative to `/Users/mdproctor/claude/casehub/slots/196/neocortex`):**

| Alias | Path |
|-------|------|
| `$API` | `cognition-api/src/main/java/io/casehub/neocortex/cognition` |
| `$API_TEST` | `cognition-api/src/test/java/io/casehub/neocortex/cognition` |
| `$COG` | `cognition/src/main/java/io/casehub/neocortex/cognition` |
| `$COG_TEST` | `cognition/src/test/java/io/casehub/neocortex/cognition` |
| `$OBS_API` | `cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability` |
| `$OBS_CORE` | `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability` |
| `$OBS_CORE_TEST` | `cognitive-observability-core/src/test/java/io/casehub/neocortex/cognitive/observability` |
| `$MI` | `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence` |

---

## Batch 1: Core API Types [journal types compile, CognitiveTrace accumulates entries in tests]

### Task 1: EventRef sealed value type

**Files:**
- Create: `$API/journal/EventRef.java`
- Test: `$API_TEST/journal/EventRefTest.java`

**Interfaces:**
- Consumes: nothing
- Produces: `EventRef` sealed interface with `Init`, `Tick(int)`, `Event(String)`, `Consolidation(EventRef)` — factory methods `init()`, `tick(int)`, `event(String)`, `consolidation(EventRef)`, `parse(String)`. `toString()` for serialization.

- [ ] **Step 1: Write failing tests**

```java
package io.casehub.neocortex.cognition.journal;

import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

class EventRefTest {

    @Test
    void initFactoryAndToString() {
        EventRef ref = EventRef.init();
        assertInstanceOf(EventRef.Init.class, ref);
        assertEquals("init", ref.toString());
    }

    @Test
    void tickFactoryAndToString() {
        EventRef ref = EventRef.tick(47);
        assertInstanceOf(EventRef.Tick.class, ref);
        assertEquals("tick:47", ref.toString());
        assertEquals(47, ((EventRef.Tick) ref).number());
    }

    @Test
    void eventFactoryAndToString() {
        EventRef ref = EventRef.event("abc123");
        assertInstanceOf(EventRef.Event.class, ref);
        assertEquals("event:abc123", ref.toString());
    }

    @Test
    void consolidationFactoryAndToString() {
        EventRef ref = EventRef.consolidation(EventRef.tick(120));
        assertInstanceOf(EventRef.Consolidation.class, ref);
        assertEquals("consolidation:tick:120", ref.toString());
    }

    @Test
    void parseRoundTrip() {
        assertEquals(EventRef.init(), EventRef.parse("init"));
        assertEquals(EventRef.tick(47), EventRef.parse("tick:47"));
        assertEquals(EventRef.event("abc"), EventRef.parse("event:abc"));
        assertEquals(EventRef.consolidation(EventRef.tick(120)),
                     EventRef.parse("consolidation:tick:120"));
    }

    @Test
    void parseInvalidThrows() {
        assertThrows(IllegalArgumentException.class, () -> EventRef.parse("garbage"));
    }

    @Test
    void equality() {
        assertEquals(EventRef.tick(1), EventRef.tick(1));
        assertNotEquals(EventRef.tick(1), EventRef.tick(2));
        assertNotEquals(EventRef.tick(1), EventRef.init());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=EventRefTest -s ../.mvn/slot-settings.xml`
Expected: compilation failure — `EventRef` does not exist

- [ ] **Step 3: Implement EventRef**

Create `$API/journal/EventRef.java`:

```java
package io.casehub.neocortex.cognition.journal;

public sealed interface EventRef {

    record Init() implements EventRef {
        @Override public String toString() { return "init"; }
    }

    record Tick(int number) implements EventRef {
        @Override public String toString() { return "tick:" + number; }
    }

    record Event(String id) implements EventRef {
        @Override public String toString() { return "event:" + id; }
    }

    record Consolidation(EventRef trigger) implements EventRef {
        @Override public String toString() { return "consolidation:" + trigger; }
    }

    static EventRef init() { return new Init(); }
    static EventRef tick(int number) { return new Tick(number); }
    static EventRef event(String id) { return new Event(id); }
    static EventRef consolidation(EventRef trigger) { return new Consolidation(trigger); }

    static EventRef parse(String s) {
        if ("init".equals(s)) return init();
        if (s.startsWith("tick:")) return tick(Integer.parseInt(s.substring(5)));
        if (s.startsWith("event:")) return event(s.substring(6));
        if (s.startsWith("consolidation:")) return consolidation(parse(s.substring(14)));
        throw new IllegalArgumentException("Unknown eventRef format: " + s);
    }
}
```

- [ ] **Step 4: Run tests — all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=EventRefTest -s ../.mvn/slot-settings.xml`
Expected: 7 tests PASS

- [ ] **Step 5: Commit**

```bash
git add cognition-api/src/main/java/io/casehub/neocortex/cognition/journal/EventRef.java \
       cognition-api/src/test/java/io/casehub/neocortex/cognition/journal/EventRefTest.java
git commit -m "feat(#961): add EventRef sealed value type with factory methods and parse/toString"
```

### Task 2: CognitiveEntry + CognitiveJournal + CognitiveTrace + type extractions

**Files:**
- Create: `$API/journal/CognitiveEntry.java`
- Create: `$API/journal/CognitiveJournal.java`
- Create: `$API/journal/CognitiveTrace.java`
- Create: `$API/journal/FacetEffect.java`
- Create: `$API/journal/CapsSettlingResult.java`
- Modify: `$OBS_API/DerivationObserver.java` — deprecate inner `FacetEffect`, delegate to new standalone
- Modify: `$API/trace/PipelineTrace.java` — deprecate inner `CapsSettlingSnapshot`, delegate to new standalone
- Modify: `cognition-api/pom.xml` — add explicit `cognitive-observability-api` dependency
- Test: `$API_TEST/journal/CognitiveTraceTest.java`
- Test: `$API_TEST/journal/CognitiveJournalTest.java`

**Interfaces:**
- Consumes: `EventRef` (Task 1), `AppraisalResult`, `EmotionRegulationResult`, `MoodBaseline`, `GraphMutation` (existing types)
- Produces: `CognitiveEntry` sealed interface (16 record types), `CognitiveJournal` interface with NOOP, `CognitiveTrace` accumulator, `FacetEffect` standalone record, `CapsSettlingResult` standalone record

- [ ] **Step 1: Add cognitive-observability-api dependency to cognition-api pom.xml**

Add to `cognition-api/pom.xml` `<dependencies>`:

```xml
<dependency>
    <groupId>io.casehub</groupId>
    <artifactId>casehub-neocortex-cognitive-observability-api</artifactId>
</dependency>
```

- [ ] **Step 2: Create FacetEffect standalone record**

Create `$API/journal/FacetEffect.java`:

```java
package io.casehub.neocortex.cognition.journal;

public record FacetEffect(String facet, double facetScore, String field, double before, double after) {
    public double delta() { return after - before; }
}
```

- [ ] **Step 3: Create CapsSettlingResult standalone record**

Create `$API/journal/CapsSettlingResult.java`:

```java
package io.casehub.neocortex.cognition.journal;

import java.util.Map;

public record CapsSettlingResult(Map<String, Double> activations, int iterations,
                                  String convergenceType, double saturationRatio) {}
```

- [ ] **Step 4: Create CognitiveEntry sealed interface with all 16 records**

Create `$API/journal/CognitiveEntry.java` with the full sealed interface from the spec (all 16 record types carrying tenantId, agentId, EventRef, timestamp plus domain-specific fields). Use the exact code from the spec's §Three Components.

- [ ] **Step 5: Create CognitiveJournal interface**

Create `$API/journal/CognitiveJournal.java`:

```java
package io.casehub.neocortex.cognition.journal;

public interface CognitiveJournal {

    CognitiveJournal NOOP = entry -> {};

    void record(CognitiveEntry entry);
}
```

- [ ] **Step 6: Create CognitiveTrace**

Create `$API/journal/CognitiveTrace.java` with the full implementation from the spec (accumulation, predicate filter, query methods: `entriesForEvent`, `entriesOfType`, `entriesForAgent`, `entries`, `clear`, `isEmpty`, `size`).

- [ ] **Step 7: Write CognitiveTrace tests**

Create `$API_TEST/journal/CognitiveTraceTest.java`:

```java
package io.casehub.neocortex.cognition.journal;

import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class CognitiveTraceTest {

    private static final String T = "tenant1";
    private static final String A = "agent1";
    private static final EventRef REF = EventRef.tick(1);
    private static final Instant NOW = Instant.now();

    @Test
    void accumulatesEntries() {
        var trace = new CognitiveTrace();
        trace.record(new CognitiveEntry.MoodShifted(T, A, REF, NOW, null, null, "pre-appraisal"));
        trace.record(new CognitiveEntry.DriveActivated(T, A, REF, NOW, List.of("aggression"), Map.of()));
        assertEquals(2, trace.size());
        assertFalse(trace.isEmpty());
    }

    @Test
    void entriesForEvent() {
        var trace = new CognitiveTrace();
        var tick1 = EventRef.tick(1);
        var tick2 = EventRef.tick(2);
        trace.record(new CognitiveEntry.MoodShifted(T, A, tick1, NOW, null, null, "t1"));
        trace.record(new CognitiveEntry.MoodShifted(T, A, tick2, NOW, null, null, "t2"));
        assertEquals(1, trace.entriesForEvent(tick1).size());
    }

    @Test
    void entriesOfType() {
        var trace = new CognitiveTrace();
        trace.record(new CognitiveEntry.MoodShifted(T, A, REF, NOW, null, null, "t"));
        trace.record(new CognitiveEntry.DriveActivated(T, A, REF, NOW, List.of(), Map.of()));
        var moods = trace.entriesOfType(CognitiveEntry.MoodShifted.class);
        assertEquals(1, moods.size());
        assertInstanceOf(CognitiveEntry.MoodShifted.class, moods.getFirst());
    }

    @Test
    void entriesForAgent() {
        var trace = new CognitiveTrace();
        trace.record(new CognitiveEntry.MoodShifted(T, "a1", REF, NOW, null, null, "t"));
        trace.record(new CognitiveEntry.MoodShifted(T, "a2", REF, NOW, null, null, "t"));
        assertEquals(1, trace.entriesForAgent("a1").size());
    }

    @Test
    void predicateFilter() {
        var trace = new CognitiveTrace(e -> e instanceof CognitiveEntry.MoodShifted);
        trace.record(new CognitiveEntry.MoodShifted(T, A, REF, NOW, null, null, "t"));
        trace.record(new CognitiveEntry.DriveActivated(T, A, REF, NOW, List.of(), Map.of()));
        assertEquals(1, trace.size());
    }

    @Test
    void clearResetsState() {
        var trace = new CognitiveTrace();
        trace.record(new CognitiveEntry.MoodShifted(T, A, REF, NOW, null, null, "t"));
        trace.clear();
        assertTrue(trace.isEmpty());
        assertEquals(0, trace.size());
    }

    @Test
    void noopRecordsNothing() {
        CognitiveJournal.NOOP.record(new CognitiveEntry.MoodShifted(T, A, REF, NOW, null, null, "t"));
        // no exception, no state — NOOP is a lambda
    }

    @Test
    void entriesReturnsUnmodifiableList() {
        var trace = new CognitiveTrace();
        trace.record(new CognitiveEntry.MoodShifted(T, A, REF, NOW, null, null, "t"));
        assertThrows(UnsupportedOperationException.class,
            () -> trace.entries().add(new CognitiveEntry.MoodShifted(T, A, REF, NOW, null, null, "x")));
    }
}
```

- [ ] **Step 8: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml test -Dtest=CognitiveTraceTest -s ../.mvn/slot-settings.xml`
Expected: 8 tests PASS

- [ ] **Step 9: Deprecate inner types and update PipelineTrace references**

Use `ide_edit_member` to deprecate `DerivationObserver.FacetEffect` (add `@Deprecated` annotation with message pointing to `io.casehub.neocortex.cognition.journal.FacetEffect`). Same for `PipelineTrace.CapsSettlingSnapshot` → `CapsSettlingResult`.

Update `DerivationTrace` and any other code referencing the inner `FacetEffect` to use the new standalone record — use `ide_find_references` to locate all usages.

- [ ] **Step 10: Build full module to verify**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition-api/pom.xml install -Dmaven.test.skip=true -s ../.mvn/slot-settings.xml`
Expected: BUILD SUCCESS

- [ ] **Step 11: Commit**

```bash
git add cognition-api/ cognitive-observability-api/
git commit -m "feat(#961): add CognitiveEntry sealed hierarchy, CognitiveJournal, CognitiveTrace, type extractions"
```

---

## Batch 2: Persistence and Routing [entries persist to store, scoping works with ThreadLocal]

### Task 3: CognitiveJournalStore interface + in-memory implementation

**Files:**
- Create: `$OBS_CORE/CognitiveJournalStore.java`
- Create: `$OBS_CORE/InMemoryCognitiveJournalStore.java`
- Test: `$OBS_CORE_TEST/CognitiveJournalStoreContractTest.java`

**Interfaces:**
- Consumes: `CognitiveEntry`, `EventRef` (Task 1-2)
- Produces: `CognitiveJournalStore` interface (`persist`, `persistAll`, `findByAgent`, `findByEventRef`, `findByTickRange`, `findByType`, `findByTimeWindow`). `InMemoryCognitiveJournalStore` implementation.

- [ ] **Step 1: Write contract test**

Create `$OBS_CORE_TEST/CognitiveJournalStoreContractTest.java`:

```java
package io.casehub.neocortex.cognitive.observability;

import io.casehub.neocortex.cognition.journal.*;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class CognitiveJournalStoreContractTest {

    private CognitiveJournalStore store;
    private static final String T = "tenant1";
    private static final String A = "agent1";
    private static final Instant NOW = Instant.now();

    @BeforeEach
    void setUp() {
        store = new InMemoryCognitiveJournalStore();
    }

    @Test
    void persistAndFindByAgent() {
        var entry = new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "test");
        store.persist(entry);
        var found = store.findByAgent(T, A);
        assertEquals(1, found.size());
        assertEquals(entry, found.getFirst());
    }

    @Test
    void findByEventRef() {
        var ref = EventRef.tick(5);
        store.persist(new CognitiveEntry.MoodShifted(T, A, ref, NOW, null, null, "match"));
        store.persist(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(6), NOW, null, null, "no-match"));
        var found = store.findByEventRef(T, A, ref);
        assertEquals(1, found.size());
    }

    @Test
    void findByTickRange() {
        store.persist(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "t1"));
        store.persist(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(5), NOW, null, null, "t5"));
        store.persist(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(10), NOW, null, null, "t10"));
        var found = store.findByTickRange(T, A, 1, 5);
        assertEquals(2, found.size());
    }

    @Test
    void findByType() {
        store.persist(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "m"));
        store.persist(new CognitiveEntry.DriveActivated(T, A, EventRef.tick(1), NOW, List.of(), Map.of()));
        var moods = store.findByType(T, A, CognitiveEntry.MoodShifted.class);
        assertEquals(1, moods.size());
    }

    @Test
    void persistAllBatch() {
        var entries = List.<CognitiveEntry>of(
            new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "a"),
            new CognitiveEntry.MoodShifted(T, A, EventRef.tick(2), NOW, null, null, "b")
        );
        store.persistAll(entries);
        assertEquals(2, store.findByAgent(T, A).size());
    }

    @Test
    void tenantIsolation() {
        store.persist(new CognitiveEntry.MoodShifted("t1", A, EventRef.tick(1), NOW, null, null, "a"));
        store.persist(new CognitiveEntry.MoodShifted("t2", A, EventRef.tick(1), NOW, null, null, "b"));
        assertEquals(1, store.findByAgent("t1", A).size());
        assertEquals(1, store.findByAgent("t2", A).size());
    }
}
```

- [ ] **Step 2: Run test — fails (classes don't exist)**

- [ ] **Step 3: Create CognitiveJournalStore interface**

Create `$OBS_CORE/CognitiveJournalStore.java` with the interface from the spec (6 query methods + `persist` + `persistAll`).

- [ ] **Step 4: Create InMemoryCognitiveJournalStore**

Create `$OBS_CORE/InMemoryCognitiveJournalStore.java`:

```java
package io.casehub.neocortex.cognitive.observability;

import io.casehub.neocortex.cognition.journal.*;
import java.time.Instant;
import java.util.*;
import java.util.concurrent.CopyOnWriteArrayList;

public class InMemoryCognitiveJournalStore implements CognitiveJournalStore {

    private final List<CognitiveEntry> entries = new CopyOnWriteArrayList<>();

    @Override
    public void persist(CognitiveEntry entry) { entries.add(entry); }

    @Override
    public void persistAll(List<CognitiveEntry> batch) { entries.addAll(batch); }

    @Override
    public List<CognitiveEntry> findByAgent(String tenantId, String agentId) {
        return entries.stream()
            .filter(e -> tenantId.equals(e.tenantId()) && agentId.equals(e.agentId()))
            .toList();
    }

    @Override
    public List<CognitiveEntry> findByEventRef(String tenantId, String agentId, EventRef eventRef) {
        return entries.stream()
            .filter(e -> tenantId.equals(e.tenantId()) && agentId.equals(e.agentId())
                         && eventRef.equals(e.eventRef()))
            .toList();
    }

    @Override
    public List<CognitiveEntry> findByTickRange(String tenantId, String agentId, int fromTick, int toTick) {
        return entries.stream()
            .filter(e -> tenantId.equals(e.tenantId()) && agentId.equals(e.agentId()))
            .filter(e -> e.eventRef() instanceof EventRef.Tick t
                         && t.number() >= fromTick && t.number() <= toTick)
            .toList();
    }

    @Override
    @SuppressWarnings("unchecked")
    public <T extends CognitiveEntry> List<T> findByType(String tenantId, String agentId, Class<T> type) {
        return entries.stream()
            .filter(e -> tenantId.equals(e.tenantId()) && agentId.equals(e.agentId()))
            .filter(type::isInstance)
            .map(type::cast)
            .toList();
    }

    @Override
    public List<CognitiveEntry> findByTimeWindow(String tenantId, String agentId, Instant from, Instant to) {
        return entries.stream()
            .filter(e -> tenantId.equals(e.tenantId()) && agentId.equals(e.agentId()))
            .filter(e -> !e.timestamp().isBefore(from) && !e.timestamp().isAfter(to))
            .toList();
    }
}
```

- [ ] **Step 5: Run tests — all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognitive-observability-core/pom.xml test -Dtest=CognitiveJournalStoreContractTest -s ../.mvn/slot-settings.xml`
Expected: 6 tests PASS

- [ ] **Step 6: Commit**

```bash
git add cognitive-observability-core/
git commit -m "feat(#961): add CognitiveJournalStore interface and InMemoryCognitiveJournalStore"
```

### Task 4: ScopedCognitiveJournal

**Files:**
- Create: `$COG/core/ScopedCognitiveJournal.java`
- Test: `$COG_TEST/core/ScopedCognitiveJournalTest.java`

**Interfaces:**
- Consumes: `CognitiveJournal` (Task 2), `CognitiveJournalStore` (Task 3), `CognitiveTrace` (Task 2)
- Produces: `ScopedCognitiveJournal` — `@ApplicationScoped` CDI bean. `beginScope(Predicate)`, `record(CognitiveEntry)`, `endScope() → CognitiveTrace`

- [ ] **Step 1: Write failing tests**

Create `$COG_TEST/core/ScopedCognitiveJournalTest.java`:

```java
package io.casehub.neocortex.cognition.core;

import io.casehub.neocortex.cognition.journal.*;
import io.casehub.neocortex.cognitive.observability.InMemoryCognitiveJournalStore;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class ScopedCognitiveJournalTest {

    private InMemoryCognitiveJournalStore store;
    private ScopedCognitiveJournal journal;
    private static final String T = "tenant1";
    private static final String A = "agent1";
    private static final Instant NOW = Instant.now();

    @BeforeEach
    void setUp() {
        store = new InMemoryCognitiveJournalStore();
        journal = new ScopedCognitiveJournal(store);
    }

    @Test
    void recordWithinScopeAccumulatesInTrace() {
        journal.beginScope(e -> true);
        journal.record(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "t"));
        CognitiveTrace trace = journal.endScope();
        assertEquals(1, trace.size());
    }

    @Test
    void recordAlsoPersistsToStore() {
        journal.beginScope(e -> true);
        journal.record(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "t"));
        journal.endScope();
        assertEquals(1, store.findByAgent(T, A).size());
    }

    @Test
    void recordWithoutScopeStillPersists() {
        journal.record(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "t"));
        assertEquals(1, store.findByAgent(T, A).size());
    }

    @Test
    void scopeFilterApplies() {
        journal.beginScope(e -> e instanceof CognitiveEntry.MoodShifted);
        journal.record(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "t"));
        journal.record(new CognitiveEntry.DriveActivated(T, A, EventRef.tick(1), NOW, List.of(), Map.of()));
        CognitiveTrace trace = journal.endScope();
        assertEquals(1, trace.size());
        assertEquals(2, store.findByAgent(T, A).size());
    }

    @Test
    void endScopeClearsThreadLocal() {
        journal.beginScope(e -> true);
        journal.record(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "t"));
        journal.endScope();
        journal.record(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(2), NOW, null, null, "no-trace"));
        // second record has no active scope — persists to store only
        assertEquals(2, store.findByAgent(T, A).size());
    }

    @Test
    void threadIsolation() throws Exception {
        journal.beginScope(e -> true);
        journal.record(new CognitiveEntry.MoodShifted(T, A, EventRef.tick(1), NOW, null, null, "main"));

        Thread other = new Thread(() -> {
            journal.beginScope(e -> true);
            journal.record(new CognitiveEntry.MoodShifted(T, "a2", EventRef.consolidation(EventRef.tick(1)),
                    NOW, null, null, "other"));
            CognitiveTrace otherTrace = journal.endScope();
            assertEquals(1, otherTrace.size());
        });
        other.start();
        other.join();

        CognitiveTrace mainTrace = journal.endScope();
        assertEquals(1, mainTrace.size());
        assertEquals("main", ((CognitiveEntry.MoodShifted) mainTrace.entries().getFirst()).trigger());
    }
}
```

- [ ] **Step 2: Run test — fails**

- [ ] **Step 3: Implement ScopedCognitiveJournal**

Create `$COG/core/ScopedCognitiveJournal.java` with the implementation from the spec: `@ApplicationScoped`, `ThreadLocal<CognitiveTrace>`, `beginScope`, `record` (dispatches to trace + store), `endScope`.

- [ ] **Step 4: Run tests — all pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=ScopedCognitiveJournalTest -s ../.mvn/slot-settings.xml`
Expected: 6 tests PASS

- [ ] **Step 5: Commit**

```bash
git add cognition/
git commit -m "feat(#961): add ScopedCognitiveJournal with ThreadLocal scope isolation"
```

---

## Batch 3: Wiring [journal injected into CognitionCore and ConsolidationScheduler, derivation bridged]

### Task 5: DerivationJournalBridge

**Files:**
- Create: `$COG/core/DerivationJournalBridge.java`
- Test: `$COG_TEST/core/DerivationJournalBridgeTest.java`

**Interfaces:**
- Consumes: `CognitiveJournal` (Task 2), `DerivationObserver` (existing L16), `EventRef` (Task 1)
- Produces: `DerivationJournalBridge implements DerivationObserver` — converts 4 callbacks to `CognitiveEntry` records

- [ ] **Step 1: Write failing tests**

Create `$COG_TEST/core/DerivationJournalBridgeTest.java`:

```java
package io.casehub.neocortex.cognition.core;

import io.casehub.neocortex.cognition.journal.*;
import io.casehub.neocortex.cognitive.observability.DerivationObserver;
import org.junit.jupiter.api.BeforeEach;
import org.junit.jupiter.api.Test;
import java.util.List;
import java.util.Map;
import static org.junit.jupiter.api.Assertions.*;

class DerivationJournalBridgeTest {

    private CognitiveTrace trace;
    private DerivationJournalBridge bridge;

    @BeforeEach
    void setUp() {
        trace = new CognitiveTrace();
        bridge = new DerivationJournalBridge(trace, "tenant1");
    }

    @Test
    void onPipelineStartProducesDerivationStarted() {
        bridge.onPipelineStart("agent1", "init",
            new DerivationObserver.PipelineInput("axes", List.of("term"), List.of("facet"), List.of("goal")));
        assertEquals(1, trace.size());
        var entry = trace.entriesOfType(CognitiveEntry.DerivationStarted.class).getFirst();
        assertEquals("agent1", entry.agentId());
        assertEquals(EventRef.init(), entry.eventRef());
        assertEquals("axes", entry.axes());
    }

    @Test
    void onStageComputationProducesStageComputed() {
        bridge.onStageComputation("agent1", "init",
            new DerivationObserver.StageComputation("method", Map.of("k", 1.0),
                List.of(new DerivationObserver.FacetEffect("f", 0.5, "field", 0.0, 0.5)),
                Map.of("k", 1.5)));
        var entry = trace.entriesOfType(CognitiveEntry.StageComputed.class).getFirst();
        assertEquals("method", entry.method());
        assertEquals(1, entry.facetEffects().size());
    }

    @Test
    void onDataFlowProducesDataFlowed() {
        bridge.onDataFlow("agent1", "init",
            new DerivationObserver.DataFlow("srcM", "srcF", "tgtM", "tgtF", 2.5));
        var entry = trace.entriesOfType(CognitiveEntry.DataFlowed.class).getFirst();
        assertEquals("srcM", entry.sourceMethod());
        assertEquals(2.5, entry.value());
    }

    @Test
    void onPipelineCompleteProducesDerivationCompleted() {
        bridge.onPipelineComplete("agent1", "init");
        assertEquals(1, trace.entriesOfType(CognitiveEntry.DerivationCompleted.class).size());
    }
}
```

- [ ] **Step 2: Run — fails**

- [ ] **Step 3: Implement DerivationJournalBridge**

Create `$COG/core/DerivationJournalBridge.java` converting each of the 4 `DerivationObserver` callbacks to the corresponding `CognitiveEntry` record. Map `DerivationObserver.FacetEffect` → `io.casehub.neocortex.cognition.journal.FacetEffect` in the conversion.

- [ ] **Step 4: Run tests — all pass**

- [ ] **Step 5: Commit**

```bash
git add cognition/
git commit -m "feat(#961): add DerivationJournalBridge — adapts DerivationObserver to CognitiveEntry"
```

### Task 6: Wire journal into CognitionCore + CognitivePipelineBuilder + ConsolidationScheduler + tick counter

**Files:**
- Modify: `$COG/core/CognitionCore.java` — add `configureCognitiveJournal(CognitiveJournal)`, move `tickCounter.incrementAndGet()` to start of `tick()`, add `beginScope`/`endScope` around tick body
- Modify: `$COG/core/CognitivePipelineBuilder.java` — accept `CognitiveJournal` via CDI, call `configureCognitiveJournal` during `build()`
- Modify: `$MI/consolidation/ConsolidationScheduler.java` — add `CognitiveJournal` `@Inject` constructor param, wrap `runPhases` with `beginScope`/`endScope`
- Test: `$COG_TEST/core/CognitionCoreJournalWiringTest.java`

**Interfaces:**
- Consumes: `CognitiveJournal` (Task 2), `ScopedCognitiveJournal` (Task 4)
- Produces: `CognitionCore.configureCognitiveJournal(CognitiveJournal)`, tick counter at start, scope lifecycle around tick()

- [ ] **Step 1: Write test for tick counter move and scope lifecycle**

Create `$COG_TEST/core/CognitionCoreJournalWiringTest.java` — verify that after `tick()`, `endScope()` returns a non-null `CognitiveTrace`. This requires constructing a minimal `CognitionCore` with a journal configured. Use the existing test patterns in the cognition module for CognitionCore setup.

- [ ] **Step 2: Modify CognitionCore**

Use `ide_insert_member` to add:
- `private CognitiveJournal cognitiveJournal = CognitiveJournal.NOOP;` field
- `public void configureCognitiveJournal(CognitiveJournal journal) { this.cognitiveJournal = journal; }` method

Use `ide_replace_member` to modify `tick()`:
- Move `tickCounter.incrementAndGet()` from line 347 to the start of the method (before any participant or direct call)
- Add `if (cognitiveJournal instanceof ScopedCognitiveJournal scoped) scoped.beginScope(e -> true);` after tick counter
- Add scope end at the method's end (before the existing PipelineSnapshotStore call)

- [ ] **Step 3: Modify CognitivePipelineBuilder**

Use `ide_insert_member` to add a `CognitiveJournal` field. Add CDI `@Inject` for it. In the `build()` method, after constructing `CognitionCore`, call `core.configureCognitiveJournal(cognitiveJournal)`.

- [ ] **Step 4: Modify ConsolidationScheduler**

Add `CognitiveJournal` to the `@Inject` constructor. Store as field. In `runPhases()`, wrap the phase execution with `beginScope`/`endScope` (same pattern as CognitionCore).

- [ ] **Step 5: Build all affected modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f pom.xml install -Dmaven.test.skip=true -pl cognition,cognition-api,mindmap-intelligence -am -s ../.mvn/slot-settings.xml`
Expected: BUILD SUCCESS

- [ ] **Step 6: Run existing tests to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -s ../.mvn/slot-settings.xml`
Expected: all existing tests PASS (journal defaults to NOOP, so no behavioral change)

- [ ] **Step 7: Commit**

```bash
git add cognition/ cognition-api/ mindmap-intelligence/
git commit -m "feat(#961): wire CognitiveJournal into CognitionCore, CognitivePipelineBuilder, ConsolidationScheduler"
```

---

## Batch 4: Instrumentation and Cleanup [journal entries recorded at each pipeline stage, old mechanisms cleaned up]

### Task 7: Instrument CognitionCore.tick() with journal entries

**Files:**
- Modify: `$COG/core/CognitionCore.java` — add `journal.record(...)` calls at 7 pipeline stages
- Test: `$COG_TEST/core/CognitionCoreJournalInstrumentationTest.java`

**Interfaces:**
- Consumes: `CognitiveJournal` (wired in Task 6), `EventRef` (Task 1), all entry types (Task 2)
- Produces: journal entries at 7 points in tick(): AttentionBriefed, MoodShifted (×2), DriveActivated (×2), AppraisalRecorded, RegulationDecided, GoalDecided, CapsSettled

- [ ] **Step 1: Write integration test**

Create `$COG_TEST/core/CognitionCoreJournalInstrumentationTest.java` — construct a CognitionCore with a `CognitiveTrace` journal (not scoped, just trace directly). Run one tick. Verify the trace contains the expected entry types in pipeline execution order:
1. AttentionBriefed
2. MoodShifted (pre-appraisal)
3. DriveActivated (first pass)
4. AppraisalRecorded
5. MoodShifted (post-appraisal)
6. DriveActivated (post-appraisal)
7. RegulationDecided (if regulation fires)
8. GoalDecided
9. CapsSettled (if CAPS runs)

Assert entry ordering matches pipeline execution order via `trace.entries()` index positions.

- [ ] **Step 2: Add journal.record() calls to CognitionCore.tick()**

At each pipeline stage in `tick()`, add the corresponding `journal.record(new CognitiveEntry.Xxx(...))` call:

| After this line | Record |
|-----------------|--------|
| `attentionMediator.drainBriefing(agentId)` | `AttentionBriefed` |
| First `tickMood(...)` call | `MoodShifted` with trigger "pre-appraisal" |
| First `drives.tick(...)` call | `DriveActivated` |
| `appraisalParticipant.currentResult(...)` | `AppraisalRecorded` |
| Second `tickMood(...)` call (post-appraisal) | `MoodShifted` with trigger "post-appraisal" |
| Second `drives.tick(...)` call (post-appraisal) | `DriveActivated` |
| `regulationParticipant.currentResult(...)` | `RegulationDecided` (guard: only if result is non-null) |
| Goals tick completion | `GoalDecided` |
| `behavioralSynthesis.lastSettlingResult()` | `CapsSettled` (guard: only if settling occurred) |

Each call uses `tenantId`, `agentId`, `EventRef.tick(tickNumber)`, `Instant.now()` plus stage-specific data from the pipeline participants.

- [ ] **Step 3: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -Dtest=CognitionCoreJournalInstrumentationTest -s ../.mvn/slot-settings.xml`
Expected: PASS

- [ ] **Step 4: Run full test suite to verify no regressions**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f cognition/pom.xml test -s ../.mvn/slot-settings.xml`
Expected: all tests PASS

- [ ] **Step 5: Commit**

```bash
git add cognition/
git commit -m "feat(#961): instrument CognitionCore.tick() — 7 runtime journal entry points"
```

### Task 8: Consolidation instrumentation + GraphMutated CDI listener + MemoryRecallObserver replacement + dead code cleanup

**Files:**
- Modify: `$MI/consolidation/ConsolidationScheduler.java` — add journal entries for consolidation phases
- Create: `$COG/core/GraphMutationJournalListener.java` — CDI `@Observes` for `GraphMutationRecorded`, writes `GraphMutated` to journal
- Modify: `$OBS_API/../../cognitive/index/CognitiveMemoryRetriever.java` — replace `MemoryRecallObserver` list with `CognitiveJournal`, record `MemoryRecalled` entries
- Delete: `$OBS_API/CompactionObserver.java` (use `ide_refactor_safe_delete`)
- Delete: `$OBS_API/ObservationCacheObserver.java` (use `ide_refactor_safe_delete`)
- Delete: `$OBS_API/MemoryRecallObserver.java` (after updating CognitiveMemoryRetriever)
- Test: `$COG_TEST/core/GraphMutationJournalListenerTest.java`

**Interfaces:**
- Consumes: `CognitiveJournal` (Task 2), `GraphMutationRecorded` CDI event (existing), `CognitiveMemoryRetriever` (existing)
- Produces: `GraphMutationJournalListener` CDI bean, updated `CognitiveMemoryRetriever` using journal

- [ ] **Step 1: Delete dead code — CompactionObserver, ObservationCacheObserver**

Use `ide_refactor_safe_delete` for both files. Verify no references exist (confirmed by review — zero references).

- [ ] **Step 2: Write GraphMutationJournalListener test**

Create `$COG_TEST/core/GraphMutationJournalListenerTest.java`:

```java
package io.casehub.neocortex.cognition.core;

import io.casehub.neocortex.cognition.journal.*;
import io.casehub.neocortex.cognitive.observability.*;
import org.junit.jupiter.api.Test;
import java.time.Instant;
import static org.junit.jupiter.api.Assertions.*;

class GraphMutationJournalListenerTest {

    @Test
    void recordsGraphMutatedOnEvent() {
        var trace = new CognitiveTrace();
        var listener = new GraphMutationJournalListener(trace);
        var mutation = new GraphMutation.NodeAdded("n1", "name", "sg1",
                null, Instant.now(), "test");
        listener.onGraphMutation(new GraphMutationRecorded("tenant1", mutation), "agent1");
        var entries = trace.entriesOfType(CognitiveEntry.GraphMutated.class);
        assertEquals(1, entries.size());
        assertEquals("tenant1", entries.getFirst().tenantId());
    }
}
```

- [ ] **Step 3: Implement GraphMutationJournalListener**

Create `$COG/core/GraphMutationJournalListener.java` — `@ApplicationScoped`, `@Observes GraphMutationRecorded` method that creates a `CognitiveEntry.GraphMutated` and records it to the journal. The agentId comes from the current scope context (or is passed as a parameter if available).

- [ ] **Step 4: Update CognitiveMemoryRetriever**

Replace the `List<MemoryRecallObserver> observers` field and constructor parameter with `CognitiveJournal journal`. Replace the loop calling `observer.onRecall(...)` with a single `journal.record(new CognitiveEntry.MemoryRecalled(...))`. Use `ide_find_references` on `MemoryRecallObserver` to verify no other consumers exist.

- [ ] **Step 5: Delete MemoryRecallObserver**

Use `ide_refactor_safe_delete` for `$OBS_API/MemoryRecallObserver.java`.

- [ ] **Step 6: Add consolidation journal entries to ConsolidationScheduler**

In `runPhases()`, after each phase completes, record `CognitiveEntry.Consolidated(...)` with phase name, input/output node counts, elapsed time.

- [ ] **Step 7: Build and test all affected modules**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn -f pom.xml install -pl cognition,cognitive-observability-api,cognitive-observability-core,cognitive-index,mindmap-intelligence -am -s ../.mvn/slot-settings.xml`
Expected: BUILD SUCCESS, all tests PASS

- [ ] **Step 8: Commit**

```bash
git add cognition/ cognitive-observability-api/ cognitive-observability-core/ cognitive-index/ mindmap-intelligence/
git commit -m "feat(#961): consolidation instrumentation, GraphMutated listener, replace MemoryRecallObserver, delete dead code"
```

## References

- [2026-11-13-runtime-cognitive-journal-design.md] — design spec this plan implements
- [decisions.md] — 7 validated design decisions (D1-D7)
- CognitionCore.tick(): `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitionCore.java:237-351`
- DerivationObserver: `cognitive-observability-api/src/main/java/io/casehub/neocortex/cognitive/observability/DerivationObserver.java`
- PipelineTrace: `cognition-api/src/main/java/io/casehub/neocortex/cognition/trace/PipelineTrace.java`
- CognitivePipelineBuilder: `cognition/src/main/java/io/casehub/neocortex/cognition/core/CognitivePipelineBuilder.java`
- ConsolidationScheduler: `mindmap-intelligence/src/main/java/io/casehub/neocortex/mindmap/intelligence/consolidation/ConsolidationScheduler.java`
- CognitiveMemoryRetriever: `cognitive-index/src/main/java/io/casehub/neocortex/cognitive/index/CognitiveMemoryRetriever.java`
- PipelineSnapshotStore: `cognitive-observability-core/src/main/java/io/casehub/neocortex/cognitive/observability/PipelineSnapshotStore.java`
- GitHub #961: casehubio/examples#961
- GitHub #960: casehubio/neocortex (derivation journal prerequisite)
