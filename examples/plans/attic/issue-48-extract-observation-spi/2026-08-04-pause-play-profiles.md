# Pause/Play/Speed Controls + Character Profile Panel — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #27 — Pause/play and speed controls for scenario playback
**Issue group:** #27, #28

**Goal:** Add transport controls (pause/play/speed) and a clickable character
profile panel to wacky-manor's scenario viewer.

**Architecture:** Server-side pause flag and speed multiplier on `WorldState`,
checked by `CharacterAgentLoop` before each LLM call. REST endpoints for
control + character profile (projected DTO, not raw descriptor). WebSocket
`control` events for multi-client sync. Lit web components for UI.

**Tech Stack:** Quarkus 3 (Java 26), quarkus-websockets-next, Lit 3, Vite, SVG

## Global Constraints

- Pre-release platform — breaking changes are free, no backward compat
- IntelliJ MCP mandatory for all `.java`/`.ts` file operations
- TDD: failing test before implementation for every Java change
- `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor` for all test runs
- Commit references issue: `feat(#27):` or `feat(#28):` prefix

---

### Task 1: WorldState pause/speed + CharacterAgentLoop integration

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/engine/WorldState.java`
- Modify: `src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java`
- Create: `src/test/java/io/casehub/examples/manor/engine/WorldStatePauseSpeedTest.java`

**Interfaces:**
- Produces: `WorldState.isPaused()`, `WorldState.setPaused(boolean)`,
  `WorldState.speedMultiplier()`, `WorldState.setSpeedMultiplier(double)`

- [ ] **Step 1: Write WorldState pause/speed tests**

```java
package io.casehub.examples.manor.engine;

import org.junit.jupiter.api.Test;
import java.util.Map;
import static org.assertj.core.api.Assertions.*;

class WorldStatePauseSpeedTest {

    private WorldState world() {
        return new WorldState(Map.of(), Map.of());
    }

    @Test
    void defaults_to_not_paused() {
        assertThat(world().isPaused()).isFalse();
    }

    @Test
    void defaults_to_speed_1x() {
        assertThat(world().speedMultiplier()).isEqualTo(1.0);
    }

    @Test
    void pause_and_resume() {
        var w = world();
        w.setPaused(true);
        assertThat(w.isPaused()).isTrue();
        w.setPaused(false);
        assertThat(w.isPaused()).isFalse();
    }

    @Test
    void speed_multiplier_set_and_get() {
        var w = world();
        w.setSpeedMultiplier(2.0);
        assertThat(w.speedMultiplier()).isEqualTo(2.0);
    }

    @Test
    void speed_clamped_low() {
        var w = world();
        w.setSpeedMultiplier(0.1);
        assertThat(w.speedMultiplier()).isEqualTo(0.25);
    }

    @Test
    void speed_clamped_high() {
        var w = world();
        w.setSpeedMultiplier(20.0);
        assertThat(w.speedMultiplier()).isEqualTo(8.0);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=WorldStatePauseSpeedTest`
Expected: compilation failure — `isPaused()`, `speedMultiplier()` not found

- [ ] **Step 3: Implement WorldState pause/speed fields**

Use `ide_insert_member` to add to `WorldState.java` after the `completionReason` field (line 27):

```java
private volatile boolean paused = false;
private volatile double speedMultiplier = 1.0;
```

And add accessors:
```java
public boolean isPaused() { return paused; }
public void setPaused(boolean paused) { this.paused = paused; }
public double speedMultiplier() { return speedMultiplier; }
public void setSpeedMultiplier(double multiplier) {
    this.speedMultiplier = Math.max(0.25, Math.min(8.0, multiplier));
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=WorldStatePauseSpeedTest`
Expected: 6 tests PASS

- [ ] **Step 5: Modify CharacterAgentLoop — pause gate + speed multiplier**

In `CharacterAgentLoop.run()`, add pause check before the LLM call (after
`sceneContext.awaitRelease()` and before `drain`). The loop already receives
`world` as a parameter.

Use `ide_replace_member` on `CharacterAgentLoop.run`:

Before the observation/LLM call block, insert:
```java
while (world.isPaused() && !world.isScenarioComplete() && character.isActive()) {
    Thread.sleep(200);
}
if (world.isScenarioComplete()) { break; }
```

Replace the `Thread.sleep(thinkDelay(character))` at the end of the loop with:
```java
Thread.sleep((long)(character.thinkDelayMs() / world.speedMultiplier()));
```

Delete the now-unused `thinkDelay` private method.

- [ ] **Step 6: Verify build compiles**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS (214+)

- [ ] **Step 7: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/engine/WorldState.java wacky-manor/src/main/java/io/casehub/examples/manor/agent/CharacterAgentLoop.java wacky-manor/src/test/java/io/casehub/examples/manor/engine/WorldStatePauseSpeedTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#27): pause/speed on WorldState, pause gate in CharacterAgentLoop"
```

---

### Task 2: WebSocket control event + REST endpoints

**Files:**
- Modify: `src/main/java/io/casehub/examples/manor/web/ManorWebSocketEvent.java`
- Modify: `src/main/java/io/casehub/examples/manor/web/ManorResource.java`
- Create: `src/test/java/io/casehub/examples/manor/web/ManorResourceControlTest.java`

**Interfaces:**
- Consumes: `WorldState.isPaused()`, `WorldState.setPaused(boolean)`,
  `WorldState.speedMultiplier()`, `WorldState.setSpeedMultiplier(double)`
- Produces: `ManorWebSocketEvent.control(String status, double speedMultiplier)`,
  `POST /manor/pause`, `POST /manor/resume`, `POST /manor/speed?rate=N`

- [ ] **Step 1: Add speedMultiplier field and control() factory to ManorWebSocketEvent**

Use `ide_edit_member` on the `ManorWebSocketEvent` record declaration to add
`Double speedMultiplier` as a 14th field (after `reason`).

Add factory method with `ide_insert_member`:
```java
public static ManorWebSocketEvent control(String status, double speedMultiplier) {
    return new ManorWebSocketEvent("control", null, null, null, null, null, status, null, null, null, null, null, null, speedMultiplier);
}
```

Update all existing factory methods to pass `null` as the 14th argument.

- [ ] **Step 2: Write ManorResourceControlTest**

```java
package io.casehub.examples.manor.web;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ManorResourceControlTest {

    @Test
    void pause_returns_404_when_no_scenario() {
        RestAssured.given()
            .when().post("/manor/pause")
            .then().statusCode(404);
    }

    @Test
    void resume_returns_404_when_no_scenario() {
        RestAssured.given()
            .when().post("/manor/resume")
            .then().statusCode(404);
    }

    @Test
    void speed_returns_404_when_no_scenario() {
        RestAssured.given()
            .queryParam("rate", 2.0)
            .when().post("/manor/speed")
            .then().statusCode(404);
    }
}
```

- [ ] **Step 3: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorResourceControlTest`
Expected: compilation failure or 404 (endpoints don't exist)

- [ ] **Step 4: Implement REST endpoints on ManorResource**

Use `ide_insert_member` to add after `getEvents()`:

```java
@POST
@Path("/pause")
public Response pauseScenario() {
    if (activeWorld == null || activeWorld.isScenarioComplete()) {
        return Response.status(Response.Status.NOT_FOUND)
                       .entity("{\"error\":\"No active scenario\"}").build();
    }
    activeWorld.setPaused(true);
    eventBus.broadcast(ManorWebSocketEvent.control("paused", activeWorld.speedMultiplier()));
    return Response.ok("{\"status\":\"paused\"}").build();
}

@POST
@Path("/resume")
public Response resumeScenario() {
    if (activeWorld == null || activeWorld.isScenarioComplete()) {
        return Response.status(Response.Status.NOT_FOUND)
                       .entity("{\"error\":\"No active scenario\"}").build();
    }
    activeWorld.setPaused(false);
    eventBus.broadcast(ManorWebSocketEvent.control("resumed", activeWorld.speedMultiplier()));
    return Response.ok("{\"status\":\"resumed\"}").build();
}

@POST
@Path("/speed")
public Response setSpeed(@jakarta.ws.rs.QueryParam("rate") double rate) {
    if (activeWorld == null || activeWorld.isScenarioComplete()) {
        return Response.status(Response.Status.NOT_FOUND)
                       .entity("{\"error\":\"No active scenario\"}").build();
    }
    activeWorld.setSpeedMultiplier(rate);
    eventBus.broadcast(ManorWebSocketEvent.control("speed", activeWorld.speedMultiplier()));
    return Response.ok("{\"status\":\"speed\",\"rate\":" + activeWorld.speedMultiplier() + "}").build();
}
```

- [ ] **Step 5: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorResourceControlTest`
Expected: 3 tests PASS

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/web/ManorWebSocketEvent.java wacky-manor/src/main/java/io/casehub/examples/manor/web/ManorResource.java wacky-manor/src/test/java/io/casehub/examples/manor/web/ManorResourceControlTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#27): control WebSocket event + pause/resume/speed REST endpoints"
```

---

### Task 3: CharacterProfileDTO + profile endpoint

**Files:**
- Create: `src/main/java/io/casehub/examples/manor/web/CharacterProfileDTO.java`
- Modify: `src/main/java/io/casehub/examples/manor/web/ManorResource.java`
- Create: `src/test/java/io/casehub/examples/manor/web/CharacterProfileDTOTest.java`
- Create: `src/test/java/io/casehub/examples/manor/web/ManorResourceProfileTest.java`

**Interfaces:**
- Consumes: `AgentRegistry.findById(String, String)`, `VocabularyRegistry.resolve(String, String)`
- Produces: `GET /manor/characters/{id}/profile` returning `CharacterProfileDTO` JSON

- [ ] **Step 1: Write CharacterProfileDTOTest**

```java
package io.casehub.examples.manor.web;

import io.casehub.eidos.api.*;
import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class CharacterProfileDTOTest {

    @Test
    void projects_basic_fields() {
        var descriptor = new AgentDescriptor(
            "test-agent", "Test Agent", null, null, null, null, null,
            null, "urn:casehub:vocab:belbin", "urn:casehub:vocab:jungian", null,
            "shaper", java.util.List.of(), AgentDisposition.builder()
                .dispositionProfile(DispositionValue.of("te", 0.35), DispositionValue.of("ni", 0.2))
                .build(),
            null, null, "test-tenancy", "A test agent briefing.",
            null, java.util.List.of(), java.util.List.of());

        var dto = CharacterProfileDTO.from(descriptor, null, null);

        assertThat(dto.agentId()).isEqualTo("test-agent");
        assertThat(dto.name()).isEqualTo("Test Agent");
        assertThat(dto.slot()).isEqualTo("shaper");
        assertThat(dto.briefing()).isEqualTo("A test agent briefing.");
        assertThat(dto.dispositionProfile()).hasSize(2);
        assertThat(dto.dispositionProfile().getFirst().term()).isEqualTo("te");
    }

    @Test
    void filters_private_goals() {
        var goals = java.util.List.of(
            new AgentGoal("public-goal", "A public goal",
                GoalPriority.PRIMARY, Visibility.PUBLIC, java.util.List.of()),
            new AgentGoal("private-goal", "A private goal",
                GoalPriority.PRIMARY, Visibility.PRIVATE, java.util.List.of()));

        var descriptor = new AgentDescriptor(
            "test", "Test", null, null, null, null, null,
            null, null, null, null,
            "shaper", java.util.List.of(), AgentDisposition.builder().build(),
            null, null, "t", null, null, goals, java.util.List.of());

        var dto = CharacterProfileDTO.from(descriptor, null, null);
        assertThat(dto.goals()).hasSize(1);
        assertThat(dto.goals().getFirst().name()).isEqualTo("public-goal");
    }

    @Test
    void filters_private_constraints() {
        var constraints = java.util.List.of(
            new AgentConstraint("public-c", "Public", Visibility.PUBLIC, ConstraintSeverity.HARD),
            new AgentConstraint("private-c", "Private", Visibility.PRIVATE, ConstraintSeverity.SOFT));

        var descriptor = new AgentDescriptor(
            "test", "Test", null, null, null, null, null,
            null, null, null, null,
            "shaper", java.util.List.of(), AgentDisposition.builder().build(),
            null, null, "t", null, null, java.util.List.of(), constraints);

        var dto = CharacterProfileDTO.from(descriptor, null, null);
        assertThat(dto.constraints()).hasSize(1);
        assertThat(dto.constraints().getFirst().name()).isEqualTo("public-c");
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CharacterProfileDTOTest`
Expected: compilation failure — `CharacterProfileDTO` not found

- [ ] **Step 3: Implement CharacterProfileDTO**

Use `ide_create_file`:

```java
package io.casehub.examples.manor.web;

import io.casehub.eidos.api.*;
import java.util.List;

public record CharacterProfileDTO(
    String agentId,
    String name,
    String slot,
    String slotLabel,
    String enneagramType,
    List<DispositionValue> dispositionProfile,
    List<CapabilityDTO> capabilities,
    List<GoalDTO> goals,
    List<ConstraintDTO> constraints,
    String briefing
) {
    public record CapabilityDTO(String name, List<String> tags) {}
    public record GoalDTO(String name, String description, String priority) {}
    public record ConstraintDTO(String name, String description, String severity) {}

    public static CharacterProfileDTO from(AgentDescriptor desc, String slotLabel, String enneagramType) {
        var caps = desc.capabilities().stream()
            .map(c -> new CapabilityDTO(c.name(), c.tags()))
            .toList();

        var goals = desc.goals().stream()
            .filter(g -> g.visibility() == Visibility.PUBLIC)
            .map(g -> new GoalDTO(g.name(), g.description(), g.priority().name()))
            .toList();

        var constraints = desc.constraints().stream()
            .filter(c -> c.visibility() == Visibility.PUBLIC)
            .map(c -> new ConstraintDTO(c.name(), c.description(), c.severity().name()))
            .toList();

        return new CharacterProfileDTO(
            desc.agentId(), desc.name(), desc.slot(), slotLabel, enneagramType,
            desc.disposition() != null ? desc.disposition().dispositionProfile() : List.of(),
            caps, goals, constraints, desc.briefing());
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=CharacterProfileDTOTest`
Expected: 3 tests PASS

- [ ] **Step 5: Write ManorResourceProfileTest**

```java
package io.casehub.examples.manor.web;

import io.quarkus.test.junit.QuarkusTest;
import io.restassured.RestAssured;
import org.junit.jupiter.api.Test;
import static org.hamcrest.Matchers.*;

@QuarkusTest
class ManorResourceProfileTest {

    @Test
    void profile_returns_descriptor_for_known_character() {
        RestAssured.given()
            .when().get("/manor/characters/penelope-pitstop/profile")
            .then()
            .statusCode(200)
            .body("agentId", equalTo("penelope-pitstop"))
            .body("name", equalTo("Penelope Pitstop"))
            .body("slot", equalTo("teamworker"))
            .body("dispositionProfile", not(empty()))
            .body("briefing", containsString("Penelope Pitstop"));
    }

    @Test
    void profile_returns_404_for_unknown_character() {
        RestAssured.given()
            .when().get("/manor/characters/nonexistent/profile")
            .then()
            .statusCode(404);
    }

    @Test
    void profile_filters_private_goals() {
        RestAssured.given()
            .when().get("/manor/characters/hooded-claw/profile")
            .then()
            .statusCode(200)
            .body("goals.findAll { it.name == 'eliminate-penelope' }", empty());
    }
}
```

- [ ] **Step 6: Implement profile endpoint on ManorResource**

Add `@Inject AgentRegistry agentRegistry;` and `@Inject VocabularyRegistry vocabRegistry;`
fields to `ManorResource`.

Use `ide_insert_member` to add after the speed endpoint:

```java
@jakarta.ws.rs.GET
@jakarta.ws.rs.Path("/characters/{id}/profile")
@jakarta.ws.rs.Produces("application/json")
public Response getCharacterProfile(@jakarta.ws.rs.PathParam("id") String id) {
    var desc = agentRegistry.findById(id, io.casehub.examples.manor.ManorConstants.TENANCY_ID);
    if (desc.isEmpty()) {
        return Response.status(Response.Status.NOT_FOUND)
                       .entity("{\"error\":\"Character not found: " + id + "\"}").build();
    }
    var descriptor = desc.get();
    String slotLabel = null;
    if (descriptor.slotVocabulary() != null && descriptor.slot() != null) {
        slotLabel = vocabRegistry.resolve(descriptor.slotVocabulary(), descriptor.slot())
                        .map(io.casehub.eidos.api.VocabularyTerm::label).orElse(descriptor.slot());
    }
    var dto = CharacterProfileDTO.from(descriptor, slotLabel, null);
    return Response.ok(dto).build();
}
```

- [ ] **Step 7: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor -Dtest=ManorResourceProfileTest`
Expected: 3 tests PASS

- [ ] **Step 8: Run full test suite**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS

- [ ] **Step 9: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/java/io/casehub/examples/manor/web/CharacterProfileDTO.java wacky-manor/src/main/java/io/casehub/examples/manor/web/ManorResource.java wacky-manor/src/test/java/io/casehub/examples/manor/web/CharacterProfileDTOTest.java wacky-manor/src/test/java/io/casehub/examples/manor/web/ManorResourceProfileTest.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#28): CharacterProfileDTO + GET /manor/characters/{id}/profile endpoint"
```

---

### Task 4: Dead code cleanup

**Files:**
- Delete: `src/main/java/io/casehub/examples/manor/CharacterProfile.java` (use `ide_refactor_safe_delete`)
- Delete: `src/main/java/io/casehub/examples/manor/CharacterProfileLoader.java` (use `ide_refactor_safe_delete`)

**Interfaces:**
- Consumes: nothing (these files have zero callers)
- Produces: nothing

- [ ] **Step 1: Safe-delete CharacterProfileLoader**

Use `ide_refactor_safe_delete` on `CharacterProfileLoader` (file level):
```
ide_refactor_safe_delete(file="src/main/java/io/casehub/examples/manor/CharacterProfileLoader.java", target_type="file")
```

- [ ] **Step 2: Safe-delete CharacterProfile**

Use `ide_refactor_safe_delete` on `CharacterProfile` (file level):
```
ide_refactor_safe_delete(file="src/main/java/io/casehub/examples/manor/CharacterProfile.java", target_type="file")
```

- [ ] **Step 3: Verify build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl wacky-manor`
Expected: all tests PASS

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add -u wacky-manor/src/main/java/io/casehub/examples/manor/CharacterProfile.java wacky-manor/src/main/java/io/casehub/examples/manor/CharacterProfileLoader.java
git -C /Users/mdproctor/claude/casehub/examples commit -m "refactor(#28): delete legacy CharacterProfile + CharacterProfileLoader (zero callers)"
```

---

### Task 5: Frontend types + transport controls

**Files:**
- Modify: `src/main/webui/src/types.ts`
- Modify: `src/main/webui/src/manor-app.ts`

**Interfaces:**
- Consumes: `POST /manor/pause`, `POST /manor/resume`, `POST /manor/speed?rate=N`,
  WebSocket `control` events
- Produces: `CharacterProfileResponse` type (for Task 7),
  `character-selected` event listener (for Task 7)

- [ ] **Step 1: Update types.ts**

Add control event to `ManorEvent` union:
```typescript
| { type: 'control'; status: 'paused' | 'resumed' | 'speed'; speedMultiplier: number }
```

Add `CharacterProfileResponse` interface:
```typescript
export interface CharacterProfileResponse {
  agentId: string;
  name: string;
  slot: string;
  slotLabel: string | null;
  enneagramType: string | null;
  dispositionProfile: Array<{ term: string; weight: number }>;
  capabilities: Array<{ name: string; tags: string[] }>;
  goals: Array<{ name: string; description: string; priority: string }>;
  constraints: Array<{ name: string; description: string; severity: string }>;
  briefing: string | null;
}
```

Add 12 new entries to `CHARACTER_COLORS`:
```typescript
'muttley': '#8B6914',
'pat-pending': '#2E8B57',
'sergeant-blast': '#556B2F',
'private-meekly': '#6B8E23',
'lazy-luke': '#DAA520',
'blubber-bear': '#8B4513',
'rock-slag': '#A0522D',
'gravel-slag': '#708090',
'rufus-ruffcut': '#B22222',
'sawtooth': '#D2691E',
'big-gruesome': '#9370DB',
'little-gruesome': '#32CD32',
```

Add 12 new entries to `CHARACTER_SHORT_NAMES`:
```typescript
'muttley': 'Muttley',
'pat-pending': 'Pat',
'sergeant-blast': 'Sgt. Blast',
'private-meekly': 'Pvt. Meekly',
'lazy-luke': 'Lazy Luke',
'blubber-bear': 'Blubber',
'rock-slag': 'Rock',
'gravel-slag': 'Gravel',
'rufus-ruffcut': 'Rufus',
'sawtooth': 'Sawtooth',
'big-gruesome': 'Big G',
'little-gruesome': 'Lil G',
```

- [ ] **Step 2: Add transport controls to manor-app.ts**

Add state properties:
```typescript
@state() private paused = false;
@state() private speed = 1.0;
@state() private selectedCharacterId: string | null = null;
```

Add `control` case to `handleEvent()`:
```typescript
case 'control':
  this.paused = event.status === 'paused';
  this.speed = event.speedMultiplier;
  break;
```

Add CSS for transport controls:
```css
.transport { display: flex; align-items: center; gap: 4px; }
.transport button { padding: 4px 10px; font-size: 12px; }
.transport button.active { background: #4a4a6a; border-color: #88a; }
.speed-group { display: flex; gap: 2px; }
```

Add transport controls to the toolbar `render()`, between status and start button:
```typescript
${this.scenarioStatus === 'running' ? html`
  <div class="transport">
    <button @click=${() => this.togglePause()}>
      ${this.paused ? '▶ Play' : '⏸ Pause'}
    </button>
    <div class="speed-group">
      ${[0.5, 1, 2, 4].map(s => html`
        <button class=${this.speed === s ? 'active' : ''}
                @click=${() => this.setSpeed(s)}>${s}x</button>
      `)}
    </div>
  </div>
` : ''}
```

Add methods:
```typescript
private async togglePause() {
  await fetch(this.paused ? '/manor/resume' : '/manor/pause', { method: 'POST' });
}

private async setSpeed(rate: number) {
  await fetch(`/manor/speed?rate=${rate}`, { method: 'POST' });
}
```

Add `character-selected` event listener in `connectedCallback()`:
```typescript
this.addEventListener('character-selected', ((e: CustomEvent) => {
  this.selectedCharacterId =
    this.selectedCharacterId === e.detail.characterId ? null : e.detail.characterId;
}) as EventListener);
```

- [ ] **Step 3: Verify Vite build**

Run: `cd /Users/mdproctor/claude/casehub/examples/wacky-manor/src/main/webui && npx vite build`
Expected: build succeeds

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/webui/src/types.ts wacky-manor/src/main/webui/src/manor-app.ts
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#27): transport controls — pause/play/speed buttons + control event handling"
```

---

### Task 6: Character sprites

**Files:**
- Modify: `src/main/webui/src/manor-view.ts`

**Interfaces:**
- Consumes: nothing new
- Produces: 12 new SVG sprites in `renderCharacterAtOrigin`

- [ ] **Step 1: Add 12 SVG sprite cases to renderCharacterAtOrigin**

Add cases to the switch statement in `renderCharacterAtOrigin()` for each
character. Each sprite follows the same pattern as existing sprites: ~20 lines
of SVG at `scale(0.55)` with `translate(${ox},${oy})`.

Characters to add: `muttley`, `pat-pending`, `sergeant-blast`, `private-meekly`,
`lazy-luke`, `blubber-bear`, `rock-slag`, `gravel-slag`, `rufus-ruffcut`,
`sawtooth`, `big-gruesome`, `little-gruesome`.

Each sprite should be recognisable at ~20px: distinctive silhouette, character-appropriate
colours from the CHARACTER_COLORS table in Task 5.

Use the Edit tool (these are inline SVG template literals, not structural Java code).

- [ ] **Step 2: Verify Vite build**

Run: `cd /Users/mdproctor/claude/casehub/examples/wacky-manor/src/main/webui && npx vite build`
Expected: build succeeds

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/webui/src/manor-view.ts
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#28): 12 character SVG sprites — all 17 characters now have distinctive visuals"
```

---

### Task 7: Character profile panel

**Files:**
- Create: `src/main/webui/src/character-profile.ts`
- Modify: `src/main/webui/src/manor-view.ts` (click handlers)
- Modify: `src/main/webui/src/manor-app.ts` (panel wiring)

**Interfaces:**
- Consumes: `GET /manor/characters/{id}/profile` → `CharacterProfileResponse`,
  `character-selected` CustomEvent, `selectedCharacterId` state from manor-app

- [ ] **Step 1: Add click handlers to manor-view.ts**

In the character rendering loop (the `charPositions.map(...)` block), wrap each
character `<g>` group with a click handler that dispatches a CustomEvent:

```typescript
<g class="char-group" style="transform: translate(${p.absX}px, ${p.baseY}px)"
   @click=${(e: Event) => { e.stopPropagation(); this.dispatchEvent(new CustomEvent('character-selected', { detail: { characterId: p.id }, bubbles: true, composed: true })); }}
   style="cursor: pointer">
```

- [ ] **Step 2: Create character-profile.ts**

New Lit component file at `src/main/webui/src/character-profile.ts`:

```typescript
import { LitElement, html, svg, css, nothing } from 'lit';
import { customElement, property, state } from 'lit/decorators.js';
import { CharacterProfileResponse, CHARACTER_COLORS } from './types.js';

@customElement('character-profile')
export class CharacterProfile extends LitElement {
  @property({ type: String }) characterId: string | null = null;
  @state() private profile: CharacterProfileResponse | null = null;
  @state() private loading = false;

  static styles = css`
    :host {
      position: absolute;
      right: 8px;
      top: 8px;
      width: 280px;
      max-height: calc(100% - 16px);
      overflow-y: auto;
      background: #1e1e32;
      border: 1px solid #444;
      border-radius: 8px;
      padding: 16px;
      color: #ddd;
      font-size: 13px;
      z-index: 10;
      box-shadow: 0 4px 20px rgba(0,0,0,0.5);
    }
    h2 { margin: 0 0 4px; font-size: 16px; color: #daa520; font-family: Georgia, serif; }
    .role { color: #999; font-size: 11px; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 12px; }
    .section { margin-bottom: 12px; }
    .section-title { font-size: 11px; font-weight: 600; color: #888; text-transform: uppercase; letter-spacing: 1px; margin-bottom: 6px; }
    .bar-chart { display: flex; flex-direction: column; gap: 3px; }
    .bar-row { display: flex; align-items: center; gap: 6px; }
    .bar-label { width: 20px; font-size: 10px; color: #aaa; text-transform: uppercase; text-align: right; }
    .bar-track { flex: 1; height: 8px; background: #2a2a3e; border-radius: 4px; overflow: hidden; }
    .bar-fill { height: 100%; border-radius: 4px; transition: width 0.3s; }
    .tag { display: inline-block; padding: 2px 6px; background: #2a2a4a; border-radius: 3px; font-size: 10px; color: #aaa; margin: 2px; }
    .goal, .constraint { padding: 4px 0; border-bottom: 1px solid #2a2a3e; }
    .goal:last-child, .constraint:last-child { border-bottom: none; }
    .priority, .severity { font-size: 10px; color: #888; }
    .briefing { font-style: italic; color: #bbb; line-height: 1.5; }
    .mbti { font-weight: 600; color: #daa520; }
    .loading { color: #666; font-style: italic; }
    .close { position: absolute; top: 8px; right: 12px; cursor: pointer; color: #666; font-size: 16px; }
    .close:hover { color: #aaa; }
  `;

  willUpdate(changed: Map<string, unknown>) {
    if (changed.has('characterId') && this.characterId) {
      this.fetchProfile();
    }
  }

  private async fetchProfile() {
    if (!this.characterId) return;
    this.loading = true;
    this.profile = null;
    try {
      const resp = await fetch(`/manor/characters/${this.characterId}/profile`);
      if (resp.ok) this.profile = await resp.json();
    } finally {
      this.loading = false;
    }
  }

  render() {
    if (!this.characterId) return nothing;
    if (this.loading) return html`<div class="loading">Loading profile...</div>`;
    if (!this.profile) return html`<div class="loading">Character not found</div>`;

    const p = this.profile;
    const color = CHARACTER_COLORS[p.agentId] || '#888';
    const maxWeight = Math.max(...p.dispositionProfile.map(d => d.weight), 0.01);

    return html`
      <span class="close" @click=${() => this.dispatchEvent(new CustomEvent('profile-close', { bubbles: true, composed: true }))}>✕</span>
      <h2 style="color: ${color}">${p.name}</h2>
      <div class="role">${p.slotLabel || p.slot}${p.enneagramType ? ` · ${p.enneagramType}` : ''}</div>

      ${p.dispositionProfile.length > 0 ? html`
        <div class="section">
          <div class="section-title">Cognitive Functions</div>
          <div class="bar-chart">
            ${p.dispositionProfile.map(d => html`
              <div class="bar-row">
                <span class="bar-label">${d.term}</span>
                <div class="bar-track">
                  <div class="bar-fill" style="width: ${(d.weight / maxWeight) * 100}%; background: ${color}"></div>
                </div>
              </div>
            `)}
          </div>
        </div>
      ` : ''}

      ${p.capabilities.length > 0 ? html`
        <div class="section">
          <div class="section-title">Capabilities</div>
          ${p.capabilities.map(c => html`
            <div>${c.name} ${c.tags.map(t => html`<span class="tag">${t}</span>`)}</div>
          `)}
        </div>
      ` : ''}

      ${p.goals.length > 0 ? html`
        <div class="section">
          <div class="section-title">Goals</div>
          ${p.goals.map(g => html`
            <div class="goal">
              ${g.description}
              <span class="priority">${g.priority}</span>
            </div>
          `)}
        </div>
      ` : ''}

      ${p.constraints.length > 0 ? html`
        <div class="section">
          <div class="section-title">Constraints</div>
          ${p.constraints.map(c => html`
            <div class="constraint">
              ${c.description}
              <span class="severity">${c.severity}</span>
            </div>
          `)}
        </div>
      ` : ''}

      ${p.briefing ? html`
        <div class="section">
          <div class="section-title">Briefing</div>
          <div class="briefing">${p.briefing}</div>
        </div>
      ` : ''}
    `;
  }
}
```

- [ ] **Step 3: Wire profile panel into manor-app.ts**

Add import: `import './character-profile.js';`

Add `profile-close` listener in `connectedCallback()`:
```typescript
this.addEventListener('profile-close', () => {
  this.selectedCharacterId = null;
});
```

Add the component in `render()` inside the `.manor-section` div, after `<manor-view>`:
```typescript
${this.selectedCharacterId ? html`
  <character-profile .characterId=${this.selectedCharacterId}></character-profile>
` : ''}
```

Add `position: relative` to `.manor-section` CSS so the absolute-positioned
profile panel anchors correctly.

- [ ] **Step 4: Verify Vite build**

Run: `cd /Users/mdproctor/claude/casehub/examples/wacky-manor/src/main/webui && npx vite build`
Expected: build succeeds

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/examples add wacky-manor/src/main/webui/src/character-profile.ts wacky-manor/src/main/webui/src/manor-view.ts wacky-manor/src/main/webui/src/manor-app.ts
git -C /Users/mdproctor/claude/casehub/examples commit -m "feat(#28): character profile panel — click sprite to reveal Eidos personality graph"
```
