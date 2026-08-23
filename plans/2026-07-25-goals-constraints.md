# Goals and Constraints Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #100 — Goals, aims, and constraints as first-class fields on AgentDescriptor
**Issue group:** #100

**Goal:** Add `AgentGoal` and `AgentConstraint` as first-class records on `AgentDescriptor` with visibility control, structural rendering in all three formats, YAML deserialization, JPA persistence, and drift detection.

**Architecture:** Six new files in `api/` (2 enums, 2 records, 2 test classes), modifications to `AgentDescriptor`, `AgentDescriptorValidator`, `AgentDescriptorComparator`, `ClasspathYamlDescriptorRegistrar`, `EidosRenderPipeline`, JPA entities + mapper, and one Flyway migration. All changes follow the existing `AgentCapability` pattern.

**Tech Stack:** Java 21, Quarkus 3.32, Jackson YAML, JPA/Hibernate, Flyway, JUnit 5, AssertJ

## Global Constraints

- IntelliJ MCP project path: `/Users/mdproctor/claude/casehub/eidos` (navigation); files edited in worktree at `/Users/mdproctor/claude/casehub/worktrees/35/eidos`
- Build: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api` (or `-pl runtime`)
- `MAX_GOAL_NAME = 100`, `MAX_GOAL_DESCRIPTION = 500`, `MAX_CONSTRAINT_NAME = 100`, `MAX_CONSTRAINT_DESCRIPTION = 500`
- `MAX_GOALS = 10`, `MAX_CONSTRAINTS = 10`
- Validation in compact constructors per protocol PP-20260530-2d6dbd
- Per-field named constants per protocol PP-20260601-347bba
- All new types in `io.casehub.eidos.api` (Tier 1, pure Java)
- Flyway next version: V7
- Pre-release — break anything for the right design

---

### Task 1: Core API Types — Visibility, GoalPriority, AgentGoal, AgentConstraint

**Files:**
- Create: `api/src/main/java/io/casehub/eidos/api/Visibility.java`
- Create: `api/src/main/java/io/casehub/eidos/api/GoalPriority.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentGoal.java`
- Create: `api/src/main/java/io/casehub/eidos/api/AgentConstraint.java`
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java` (add constants)
- Test: `api/src/test/java/io/casehub/eidos/api/AgentGoalTest.java`
- Test: `api/src/test/java/io/casehub/eidos/api/AgentConstraintTest.java`

**Interfaces:**
- Produces: `Visibility { PUBLIC, PRIVATE }`, `GoalPriority { PRIMARY, SECONDARY }`
- Produces: `AgentGoal(String name, String description, GoalPriority priority, Visibility visibility)`
- Produces: `AgentConstraint(String name, String description, Visibility visibility)`
- Produces: `AgentDescriptorValidator.MAX_GOAL_NAME`, `MAX_GOAL_DESCRIPTION`, `MAX_CONSTRAINT_NAME`, `MAX_CONSTRAINT_DESCRIPTION`

- [ ] **Step 1: Write AgentGoal and AgentConstraint tests**

Create `api/src/test/java/io/casehub/eidos/api/AgentGoalTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentGoalTest {

    @Test
    void valid_goal_constructs_successfully() {
        var goal = new AgentGoal("find-diamond", "Find the Doily Diamond",
            GoalPriority.PRIMARY, Visibility.PUBLIC);
        assertThat(goal.name()).isEqualTo("find-diamond");
        assertThat(goal.description()).isEqualTo("Find the Doily Diamond");
        assertThat(goal.priority()).isEqualTo(GoalPriority.PRIMARY);
        assertThat(goal.visibility()).isEqualTo(Visibility.PUBLIC);
    }

    @Test void null_name_throws() {
        assertThatThrownBy(() -> new AgentGoal(null, "desc", GoalPriority.PRIMARY, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName()).isEqualTo("goal.name"));
    }

    @Test void blank_name_throws() {
        assertThatThrownBy(() -> new AgentGoal("  ", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void name_exceeds_100_throws() {
        assertThatThrownBy(() -> new AgentGoal("a".repeat(101), "desc", GoalPriority.PRIMARY, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void name_at_100_accepted() {
        assertThatNoException().isThrownBy(
            () -> new AgentGoal("a".repeat(100), "desc", GoalPriority.PRIMARY, Visibility.PUBLIC));
    }

    @Test void null_description_throws() {
        assertThatThrownBy(() -> new AgentGoal("g", null, GoalPriority.PRIMARY, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName()).isEqualTo("goal.description"));
    }

    @Test void description_exceeds_500_throws() {
        assertThatThrownBy(() -> new AgentGoal("g", "d".repeat(501), GoalPriority.PRIMARY, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void null_priority_throws() {
        assertThatThrownBy(() -> new AgentGoal("g", "d", null, Visibility.PUBLIC))
            .isInstanceOf(NullPointerException.class);
    }

    @Test void null_visibility_throws() {
        assertThatThrownBy(() -> new AgentGoal("g", "d", GoalPriority.PRIMARY, null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test void name_with_control_char_throws() {
        assertThatThrownBy(() -> new AgentGoal("goal\ttab", "desc", GoalPriority.PRIMARY, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void description_with_bidi_control_throws() {
        assertThatThrownBy(() -> new AgentGoal("g", "desc‏hidden", GoalPriority.PRIMARY, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }
}
```

Create `api/src/test/java/io/casehub/eidos/api/AgentConstraintTest.java`:

```java
package io.casehub.eidos.api;

import org.junit.jupiter.api.Test;
import static org.assertj.core.api.Assertions.*;

class AgentConstraintTest {

    @Test
    void valid_constraint_constructs_successfully() {
        var c = new AgentConstraint("never-break-cover",
            "Never reveal your true identity", Visibility.PRIVATE);
        assertThat(c.name()).isEqualTo("never-break-cover");
        assertThat(c.description()).isEqualTo("Never reveal your true identity");
        assertThat(c.visibility()).isEqualTo(Visibility.PRIVATE);
    }

    @Test void null_name_throws() {
        assertThatThrownBy(() -> new AgentConstraint(null, "desc", Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName()).isEqualTo("constraint.name"));
    }

    @Test void blank_name_throws() {
        assertThatThrownBy(() -> new AgentConstraint("", "desc", Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void name_exceeds_100_throws() {
        assertThatThrownBy(() -> new AgentConstraint("c".repeat(101), "desc", Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void null_description_throws() {
        assertThatThrownBy(() -> new AgentConstraint("c", null, Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class)
            .satisfies(ex -> assertThat(((AgentValidationException) ex).fieldName()).isEqualTo("constraint.description"));
    }

    @Test void description_exceeds_500_throws() {
        assertThatThrownBy(() -> new AgentConstraint("c", "d".repeat(501), Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }

    @Test void null_visibility_throws() {
        assertThatThrownBy(() -> new AgentConstraint("c", "d", null))
            .isInstanceOf(NullPointerException.class);
    }

    @Test void name_with_control_char_throws() {
        assertThatThrownBy(() -> new AgentConstraint("con\nstraint", "desc", Visibility.PUBLIC))
            .isInstanceOf(AgentValidationException.class);
    }
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="AgentGoalTest,AgentConstraintTest"`
Expected: FAIL — classes do not exist yet

- [ ] **Step 3: Create the enums and records**

Create `api/src/main/java/io/casehub/eidos/api/Visibility.java`:

```java
package io.casehub.eidos.api;

public enum Visibility { PUBLIC, PRIVATE }
```

Create `api/src/main/java/io/casehub/eidos/api/GoalPriority.java`:

```java
package io.casehub.eidos.api;

public enum GoalPriority { PRIMARY, SECONDARY }
```

Add constants to `AgentDescriptorValidator.java` (after `MAX_DESCRIPTION`):

```java
static final int MAX_GOAL_NAME              = 100;
static final int MAX_GOAL_DESCRIPTION       = 500;
static final int MAX_CONSTRAINT_NAME        = 100;
static final int MAX_CONSTRAINT_DESCRIPTION = 500;
static final int MAX_GOALS                  = 10;
static final int MAX_CONSTRAINTS            = 10;
```

Create `api/src/main/java/io/casehub/eidos/api/AgentGoal.java`:

```java
package io.casehub.eidos.api;

import java.util.Objects;

public record AgentGoal(
        String name,
        String description,
        GoalPriority priority,
        Visibility visibility
) {
    public AgentGoal {
        AgentDescriptorValidator.validateRequired("goal.name", name,
            AgentDescriptorValidator.MAX_GOAL_NAME);
        AgentDescriptorValidator.validateRequired("goal.description", description,
            AgentDescriptorValidator.MAX_GOAL_DESCRIPTION);
        Objects.requireNonNull(priority, "goal.priority must not be null");
        Objects.requireNonNull(visibility, "goal.visibility must not be null");
    }
}
```

Create `api/src/main/java/io/casehub/eidos/api/AgentConstraint.java`:

```java
package io.casehub.eidos.api;

import java.util.Objects;

public record AgentConstraint(
        String name,
        String description,
        Visibility visibility
) {
    public AgentConstraint {
        AgentDescriptorValidator.validateRequired("constraint.name", name,
            AgentDescriptorValidator.MAX_CONSTRAINT_NAME);
        AgentDescriptorValidator.validateRequired("constraint.description", description,
            AgentDescriptorValidator.MAX_CONSTRAINT_DESCRIPTION);
        Objects.requireNonNull(visibility, "constraint.visibility must not be null");
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="AgentGoalTest,AgentConstraintTest"`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/Visibility.java \
       api/src/main/java/io/casehub/eidos/api/GoalPriority.java \
       api/src/main/java/io/casehub/eidos/api/AgentGoal.java \
       api/src/main/java/io/casehub/eidos/api/AgentConstraint.java \
       api/src/test/java/io/casehub/eidos/api/AgentGoalTest.java \
       api/src/test/java/io/casehub/eidos/api/AgentConstraintTest.java \
       api/src/main/java/io/casehub/eidos/api/AgentDescriptorValidator.java
git commit -m "feat(#100): add AgentGoal, AgentConstraint, Visibility, GoalPriority types

Refs #100"
```

---

### Task 2: AgentDescriptor — Goals and Constraints Fields

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java` (add fields, builder, convenience methods, validation)
- Test: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java` (add cases)

**Interfaces:**
- Consumes: `AgentGoal`, `AgentConstraint`, `Visibility` from Task 1
- Produces: `AgentDescriptor.goals()` → `List<AgentGoal>`, `AgentDescriptor.constraints()` → `List<AgentConstraint>`, `AgentDescriptor.publicGoals()`, `AgentDescriptor.publicConstraints()`, `Builder.goals(List<AgentGoal>)`, `Builder.constraints(List<AgentConstraint>)`

- [ ] **Step 1: Write failing tests for descriptor-level goal/constraint validation**

Add to `AgentDescriptorValidatorTest.java`:

```java
@Test
void goals_default_to_empty_list() {
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").build();
    assertThat(d.goals()).isEmpty();
    assertThat(d.constraints()).isEmpty();
}

@Test
void duplicate_goal_names_throws() {
    var goals = List.of(
        new AgentGoal("find-diamond", "Find it", GoalPriority.PRIMARY, Visibility.PUBLIC),
        new AgentGoal("find-diamond", "Find it again", GoalPriority.SECONDARY, Visibility.PUBLIC));
    assertThatThrownBy(() -> AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").goals(goals).build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("find-diamond");
}

@Test
void duplicate_constraint_names_throws() {
    var constraints = List.of(
        new AgentConstraint("no-violence", "No violence", Visibility.PUBLIC),
        new AgentConstraint("no-violence", "Avoid violence", Visibility.PUBLIC));
    assertThatThrownBy(() -> AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").constraints(constraints).build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("no-violence");
}

@Test
void goals_exceeding_max_throws() {
    var goals = java.util.stream.IntStream.rangeClosed(1, 11)
        .mapToObj(i -> new AgentGoal("g-" + i, "desc", GoalPriority.PRIMARY, Visibility.PUBLIC))
        .toList();
    assertThatThrownBy(() -> AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").goals(goals).build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("goals");
}

@Test
void constraints_exceeding_max_throws() {
    var constraints = java.util.stream.IntStream.rangeClosed(1, 11)
        .mapToObj(i -> new AgentConstraint("c-" + i, "desc", Visibility.PUBLIC))
        .toList();
    assertThatThrownBy(() -> AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").constraints(constraints).build())
        .isInstanceOf(AgentValidationException.class)
        .hasMessageContaining("constraints");
}

@Test
void publicGoals_filters_by_visibility() {
    var goals = List.of(
        new AgentGoal("public-goal", "Visible", GoalPriority.PRIMARY, Visibility.PUBLIC),
        new AgentGoal("private-goal", "Hidden", GoalPriority.SECONDARY, Visibility.PRIVATE));
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").goals(goals).build();
    assertThat(d.publicGoals()).hasSize(1);
    assertThat(d.publicGoals().get(0).name()).isEqualTo("public-goal");
}

@Test
void publicConstraints_filters_by_visibility() {
    var constraints = List.of(
        new AgentConstraint("public-c", "Visible", Visibility.PUBLIC),
        new AgentConstraint("private-c", "Hidden", Visibility.PRIVATE));
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").constraints(constraints).build();
    assertThat(d.publicConstraints()).hasSize(1);
    assertThat(d.publicConstraints().get(0).name()).isEqualTo("public-c");
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="AgentDescriptorValidatorTest"`
Expected: FAIL — goals()/constraints()/publicGoals()/publicConstraints() don't exist

- [ ] **Step 3: Modify AgentDescriptor**

Add two fields after `briefing` in the record declaration:

```java
List<AgentGoal> goals,
List<AgentConstraint> constraints
```

Add to compact constructor (after the briefing validation):

```java
goals = goals != null ? List.copyOf(goals) : List.of();
constraints = constraints != null ? List.copyOf(constraints) : List.of();
if (goals.size() > AgentDescriptorValidator.MAX_GOALS) {
    throw new AgentValidationException("goals",
        "exceeds maximum count " + AgentDescriptorValidator.MAX_GOALS + " (was " + goals.size() + ")");
}
if (constraints.size() > AgentDescriptorValidator.MAX_CONSTRAINTS) {
    throw new AgentValidationException("constraints",
        "exceeds maximum count " + AgentDescriptorValidator.MAX_CONSTRAINTS + " (was " + constraints.size() + ")");
}
if (goals.size() > 1) {
    long distinctNames = goals.stream().map(AgentGoal::name).distinct().count();
    if (distinctNames < goals.size()) {
        String dup = goals.stream().map(AgentGoal::name)
            .collect(java.util.stream.Collectors.groupingBy(n -> n, java.util.stream.Collectors.counting()))
            .entrySet().stream().filter(e -> e.getValue() > 1).map(java.util.Map.Entry::getKey)
            .findFirst().orElse("?");
        throw new AgentValidationException("goals", "duplicate goal name: " + dup);
    }
}
if (constraints.size() > 1) {
    long distinctNames = constraints.stream().map(AgentConstraint::name).distinct().count();
    if (distinctNames < constraints.size()) {
        String dup = constraints.stream().map(AgentConstraint::name)
            .collect(java.util.stream.Collectors.groupingBy(n -> n, java.util.stream.Collectors.counting()))
            .entrySet().stream().filter(e -> e.getValue() > 1).map(java.util.Map.Entry::getKey)
            .findFirst().orElse("?");
        throw new AgentValidationException("constraints", "duplicate constraint name: " + dup);
    }
}
```

Add convenience methods after `vocabUriForAxis`:

```java
public List<AgentGoal> publicGoals() {
    return goals.stream().filter(g -> g.visibility() == Visibility.PUBLIC).toList();
}

public List<AgentConstraint> publicConstraints() {
    return constraints.stream().filter(c -> c.visibility() == Visibility.PUBLIC).toList();
}
```

Add to Builder — new fields:

```java
private List<AgentGoal> goals;
private List<AgentConstraint> constraints;
```

Add builder methods:

```java
public Builder goals(List<AgentGoal> v)             { this.goals       = v; return this; }
public Builder constraints(List<AgentConstraint> v)  { this.constraints = v; return this; }
```

Update `build()` to include new fields:

```java
return new AgentDescriptor(
    agentId, name, version, provider,
    modelFamily, modelVersion, weightsFingerprint,
    domainVocabulary, slotVocabulary, dispositionVocabulary,
    axisVocabularies, slot, capabilities, disposition,
    jurisdiction, dataHandlingPolicy, tenancyId, briefing,
    goals, constraints);
```

- [ ] **Step 4: Run full api module tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS (all existing tests + new tests). Existing tests pass because builder defaults goals/constraints to null → List.of(). The `AgentDescriptorMapper.toRecord()` call in runtime/ will fail, but that's a separate module.

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/AgentDescriptor.java \
       api/src/test/java/io/casehub/eidos/api/AgentDescriptorValidatorTest.java
git commit -m "feat(#100): add goals and constraints fields to AgentDescriptor

Refs #100"
```

---

### Task 3: AgentDescriptorComparator — Drift Detection

**Files:**
- Modify: `api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java`
- Modify: `api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor.goals()`, `AgentDescriptor.constraints()`, `AgentGoal`, `AgentConstraint` from Tasks 1-2
- Produces: `compareGoals()`, `compareConstraints()` methods; updated `COMPARED_FIELD_COUNT`, `COMPARED_GOAL_FIELD_COUNT`, `COMPARED_CONSTRAINT_FIELD_COUNT`

- [ ] **Step 1: Write failing tests**

Add to `AgentDescriptorComparatorTest.java`:

```java
@Test
void comparatorCoversAllGoalComponents() {
    int total = AgentGoal.class.getRecordComponents().length;
    int matchKey = 1; // name
    assertThat(AgentDescriptorComparator.COMPARED_GOAL_FIELD_COUNT).isEqualTo(total - matchKey);
}

@Test
void comparatorCoversAllConstraintComponents() {
    int total = AgentConstraint.class.getRecordComponents().length;
    int matchKey = 1; // name
    assertThat(AgentDescriptorComparator.COMPARED_CONSTRAINT_FIELD_COUNT).isEqualTo(total - matchKey);
}

@Test
void goal_added() {
    var desired = withField(b -> b.goals(List.of(
        new AgentGoal("find-diamond", "Find it", GoalPriority.PRIMARY, Visibility.PUBLIC))));
    var actual = base();
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("goals[find-diamond]")
        && d.desiredValue().equals("(present)") && d.actualValue().equals("(absent)"));
}

@Test
void goal_removed() {
    var desired = base();
    var actual = withField(b -> b.goals(List.of(
        new AgentGoal("find-diamond", "Find it", GoalPriority.PRIMARY, Visibility.PUBLIC))));
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("goals[find-diamond]")
        && d.desiredValue().equals("(absent)") && d.actualValue().equals("(present)"));
}

@Test
void goal_description_drifted() {
    var g1 = new AgentGoal("g", "Old", GoalPriority.PRIMARY, Visibility.PUBLIC);
    var g2 = new AgentGoal("g", "New", GoalPriority.PRIMARY, Visibility.PUBLIC);
    var desired = withField(b -> b.goals(List.of(g1)));
    var actual  = withField(b -> b.goals(List.of(g2)));
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("goals[g].description"));
}

@Test
void goal_priority_drifted() {
    var g1 = new AgentGoal("g", "d", GoalPriority.PRIMARY, Visibility.PUBLIC);
    var g2 = new AgentGoal("g", "d", GoalPriority.SECONDARY, Visibility.PUBLIC);
    var desired = withField(b -> b.goals(List.of(g1)));
    var actual  = withField(b -> b.goals(List.of(g2)));
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("goals[g].priority"));
}

@Test
void goal_visibility_drifted() {
    var g1 = new AgentGoal("g", "d", GoalPriority.PRIMARY, Visibility.PUBLIC);
    var g2 = new AgentGoal("g", "d", GoalPriority.PRIMARY, Visibility.PRIVATE);
    var desired = withField(b -> b.goals(List.of(g1)));
    var actual  = withField(b -> b.goals(List.of(g2)));
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("goals[g].visibility"));
}

@Test
void constraint_added() {
    var desired = withField(b -> b.constraints(List.of(
        new AgentConstraint("no-violence", "desc", Visibility.PUBLIC))));
    var actual = base();
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("constraints[no-violence]")
        && d.desiredValue().equals("(present)"));
}

@Test
void constraint_description_drifted() {
    var c1 = new AgentConstraint("c", "Old", Visibility.PUBLIC);
    var c2 = new AgentConstraint("c", "New", Visibility.PUBLIC);
    var desired = withField(b -> b.constraints(List.of(c1)));
    var actual  = withField(b -> b.constraints(List.of(c2)));
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("constraints[c].description"));
}

@Test
void constraint_visibility_drifted() {
    var c1 = new AgentConstraint("c", "d", Visibility.PUBLIC);
    var c2 = new AgentConstraint("c", "d", Visibility.PRIVATE);
    var desired = withField(b -> b.constraints(List.of(c1)));
    var actual  = withField(b -> b.constraints(List.of(c2)));
    var result = AgentDescriptorComparator.compare(desired, actual);
    assertThat(result.drifts()).anyMatch(d -> d.field().equals("constraints[c].visibility"));
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api -Dtest="AgentDescriptorComparatorTest"`
Expected: FAIL — `COMPARED_GOAL_FIELD_COUNT` not defined; `comparatorCoversAllDescriptorComponents` will also fail because `COMPARED_FIELD_COUNT` is stale

- [ ] **Step 3: Implement comparator changes**

Update constants:

```java
static final int COMPARED_FIELD_COUNT = 18;  // was 16
static final int COMPARED_GOAL_FIELD_COUNT = 3;
static final int COMPARED_CONSTRAINT_FIELD_COUNT = 2;
```

In `compare()`, add before `return`:

```java
compareGoals(drifts, desired.goals(), actual.goals());
compareConstraints(drifts, desired.constraints(), actual.constraints());
```

Add methods:

```java
private static void compareGoals(List<FieldDrift> drifts,
                                  List<AgentGoal> desired,
                                  List<AgentGoal> actual) {
    Map<String, AgentGoal> desiredByName = desired.stream()
            .collect(Collectors.toMap(AgentGoal::name, g -> g));
    Map<String, AgentGoal> actualByName = actual.stream()
            .collect(Collectors.toMap(AgentGoal::name, g -> g));
    for (String name : new TreeSet<>(desiredByName.keySet())) {
        if (!actualByName.containsKey(name)) {
            drifts.add(new FieldDrift("goals[" + name + "]", "(present)", "(absent)"));
        }
    }
    for (String name : new TreeSet<>(actualByName.keySet())) {
        if (!desiredByName.containsKey(name)) {
            drifts.add(new FieldDrift("goals[" + name + "]", "(absent)", "(present)"));
        }
    }
    for (var entry : new TreeMap<>(desiredByName).entrySet()) {
        AgentGoal actualGoal = actualByName.get(entry.getKey());
        if (actualGoal != null) {
            String prefix = "goals[" + entry.getKey() + "].";
            compareField(drifts, prefix + "description", entry.getValue().description(), actualGoal.description());
            compareField(drifts, prefix + "priority", entry.getValue().priority(), actualGoal.priority());
            compareField(drifts, prefix + "visibility", entry.getValue().visibility(), actualGoal.visibility());
        }
    }
}

private static void compareConstraints(List<FieldDrift> drifts,
                                        List<AgentConstraint> desired,
                                        List<AgentConstraint> actual) {
    Map<String, AgentConstraint> desiredByName = desired.stream()
            .collect(Collectors.toMap(AgentConstraint::name, c -> c));
    Map<String, AgentConstraint> actualByName = actual.stream()
            .collect(Collectors.toMap(AgentConstraint::name, c -> c));
    for (String name : new TreeSet<>(desiredByName.keySet())) {
        if (!actualByName.containsKey(name)) {
            drifts.add(new FieldDrift("constraints[" + name + "]", "(present)", "(absent)"));
        }
    }
    for (String name : new TreeSet<>(actualByName.keySet())) {
        if (!desiredByName.containsKey(name)) {
            drifts.add(new FieldDrift("constraints[" + name + "]", "(absent)", "(present)"));
        }
    }
    for (var entry : new TreeMap<>(desiredByName).entrySet()) {
        AgentConstraint actualC = actualByName.get(entry.getKey());
        if (actualC != null) {
            String prefix = "constraints[" + entry.getKey() + "].";
            compareField(drifts, prefix + "description", entry.getValue().description(), actualC.description());
            compareField(drifts, prefix + "visibility", entry.getValue().visibility(), actualC.visibility());
        }
    }
}
```

- [ ] **Step 4: Run full api tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl api`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add api/src/main/java/io/casehub/eidos/api/AgentDescriptorComparator.java \
       api/src/test/java/io/casehub/eidos/api/AgentDescriptorComparatorTest.java
git commit -m "feat(#100): add goal/constraint drift detection to AgentDescriptorComparator

Refs #100"
```

---

### Task 4: YAML Deserialization

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java`

**Interfaces:**
- Consumes: `AgentGoal`, `AgentConstraint`, `GoalPriority`, `Visibility` from Task 1
- Produces: YAML → `AgentDescriptor` with goals and constraints populated

- [ ] **Step 1: Write failing tests**

Add to `ClasspathYamlDescriptorRegistrarTest.java`:

```java
@Test
void goals_and_constraints_deserialized() {
    var yaml = """
        descriptors:
          - agentId: hooded-claw
            name: The Hooded Claw
            slot: villain
            tenancyId: wacky-manor
            goals:
              - name: eliminate-penelope
                description: "Kill Penelope Pitstop"
                priority: PRIMARY
                visibility: PRIVATE
              - name: win-treasure
                description: "Win the treasure hunt"
                priority: SECONDARY
                visibility: PUBLIC
            constraints:
              - name: never-break-cover
                description: "Never reveal your true identity"
                visibility: PRIVATE
              - name: elaborate-schemes
                description: "Schemes must be elaborate"
                visibility: PUBLIC
        """;
    var result = parse(yaml);
    assertThat(result).hasSize(1);
    var d = result.get(0);
    assertThat(d.goals()).hasSize(2);
    assertThat(d.goals().get(0).name()).isEqualTo("eliminate-penelope");
    assertThat(d.goals().get(0).priority()).isEqualTo(io.casehub.eidos.api.GoalPriority.PRIMARY);
    assertThat(d.goals().get(0).visibility()).isEqualTo(io.casehub.eidos.api.Visibility.PRIVATE);
    assertThat(d.goals().get(1).priority()).isEqualTo(io.casehub.eidos.api.GoalPriority.SECONDARY);
    assertThat(d.constraints()).hasSize(2);
    assertThat(d.constraints().get(0).name()).isEqualTo("never-break-cover");
    assertThat(d.constraints().get(0).visibility()).isEqualTo(io.casehub.eidos.api.Visibility.PRIVATE);
}

@Test
void missing_goals_and_constraints_defaults_to_empty() {
    var yaml = """
        descriptors:
          - agentId: minimal
            name: Minimal
            slot: s
            tenancyId: t
        """;
    var result = parse(yaml);
    assertThat(result.get(0).goals()).isEmpty();
    assertThat(result.get(0).constraints()).isEmpty();
}

@Test
void goal_missing_visibility_throws() {
    var yaml = """
        descriptors:
          - agentId: bad
            name: N
            slot: s
            tenancyId: t
            goals:
              - name: g
                description: d
                priority: PRIMARY
        """;
    assertThatThrownBy(() -> parse(yaml))
        .isInstanceOf(NullPointerException.class);
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ClasspathYamlDescriptorRegistrarTest"`
Expected: FAIL — goals/constraints not mapped

- [ ] **Step 3: Add GoalConfig, ConstraintConfig, and mapping logic**

Add inner classes to `ClasspathYamlDescriptorRegistrar`:

```java
static class GoalConfig {
    public String name;
    public String description;
    public GoalPriority priority;
    public Visibility visibility;
}

static class ConstraintConfig {
    public String name;
    public String description;
    public Visibility visibility;
}
```

Add to `DescriptorConfig`:

```java
public List<GoalConfig> goals;
public List<ConstraintConfig> constraints;
```

Add to `toDescriptor()` (before `return builder.build()`):

```java
if (cfg.goals != null) {
    builder.goals(cfg.goals.stream().map(g ->
        new AgentGoal(g.name, g.description, g.priority, g.visibility)
    ).toList());
}

if (cfg.constraints != null) {
    builder.constraints(cfg.constraints.stream().map(c ->
        new AgentConstraint(c.name, c.description, c.visibility)
    ).toList());
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="ClasspathYamlDescriptorRegistrarTest"`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrar.java \
       runtime/src/test/java/io/casehub/eidos/runtime/registrar/ClasspathYamlDescriptorRegistrarTest.java
git commit -m "feat(#100): YAML deserialization for goals and constraints

Refs #100"
```

---

### Task 5: Rendering Pipeline

**Files:**
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java`
- Modify: `runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java`

**Interfaces:**
- Consumes: `AgentDescriptor.goals()`, `AgentDescriptor.constraints()`, `AgentDescriptor.publicGoals()`, `AgentDescriptor.publicConstraints()`, `AgentGoal`, `AgentConstraint`, `GoalPriority`, `Visibility` from Tasks 1-2
- Produces: Goals and constraints sections in MARKDOWN, PROSE, and A2A_CARD output; goals/constraints in `buildDescriptorPayload()` for cache key correctness

- [ ] **Step 1: Write failing tests**

Add to `EidosRenderPipelineTest.java`:

```java
static AgentDescriptor descriptorWithGoalsAndConstraints() {
    return AgentDescriptor.builder()
        .agentId("hooded-claw").name("The Hooded Claw").slot("villain").tenancyId("wacky-manor")
        .goals(List.of(
            new AgentGoal("win-treasure", "Win the treasure hunt",
                GoalPriority.SECONDARY, Visibility.PUBLIC),
            new AgentGoal("eliminate-penelope", "Kill Penelope Pitstop",
                GoalPriority.PRIMARY, Visibility.PRIVATE)))
        .constraints(List.of(
            new AgentConstraint("elaborate-schemes", "Schemes must be elaborate", Visibility.PUBLIC),
            new AgentConstraint("never-break-cover", "Never reveal your true identity", Visibility.PRIVATE)))
        .build();
}

@Test
void markdown_renders_objectives_section_sorted_by_priority_then_name() {
    var d = descriptorWithGoalsAndConstraints();
    var ctx = AgentPromptContext.forFormat(MARKDOWN);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).contains("## Objectives");
    int primaryIdx = result.content().indexOf("**[PRIMARY]** Kill Penelope");
    int secondaryIdx = result.content().indexOf("**[SECONDARY]** Win the treasure");
    assertThat(primaryIdx).isLessThan(secondaryIdx);
}

@Test
void markdown_renders_constraints_section_sorted_alphabetically() {
    var d = descriptorWithGoalsAndConstraints();
    var ctx = AgentPromptContext.forFormat(MARKDOWN);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).contains("## Constraints");
    int elaborateIdx = result.content().indexOf("elaborate-schemes");
    int neverIdx = result.content().indexOf("never-break-cover");
    assertThat(elaborateIdx).isLessThan(neverIdx);
}

@Test
void markdown_objectives_before_disposition() {
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t")
        .goals(List.of(new AgentGoal("g", "d", GoalPriority.PRIMARY, Visibility.PUBLIC)))
        .disposition(AgentDisposition.builder().autonomy("high").build())
        .build();
    var ctx = AgentPromptContext.forFormat(MARKDOWN);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    int objectivesIdx = result.content().indexOf("## Objectives");
    int dispositionIdx = result.content().indexOf("## How You Operate");
    assertThat(objectivesIdx).isLessThan(dispositionIdx);
}

@Test
void prose_renders_goals_and_constraints() {
    var d = descriptorWithGoalsAndConstraints();
    var ctx = AgentPromptContext.forFormat(PROSE);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).containsIgnoringCase("objective");
    assertThat(result.content()).contains("Kill Penelope");
    assertThat(result.content()).contains("Schemes must be elaborate");
}

@Test
void a2a_card_excludes_private_goals_and_constraints() {
    var d = descriptorWithGoalsAndConstraints();
    var ctx = AgentPromptContext.forFormat(A2A_CARD);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).contains("win-treasure");
    assertThat(result.content()).doesNotContain("eliminate-penelope");
    assertThat(result.content()).contains("elaborate-schemes");
    assertThat(result.content()).doesNotContain("never-break-cover");
}

@Test
void empty_goals_omits_objectives_section() {
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").build();
    var ctx = AgentPromptContext.forFormat(MARKDOWN);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).doesNotContain("Objectives");
}

@Test
void empty_constraints_omits_constraints_section() {
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t").build();
    var ctx = AgentPromptContext.forFormat(MARKDOWN);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).doesNotContain("Constraints");
}

@Test
void a2a_card_omits_goals_key_when_no_public_goals() {
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t")
        .goals(List.of(new AgentGoal("secret", "d", GoalPriority.PRIMARY, Visibility.PRIVATE)))
        .build();
    var ctx = AgentPromptContext.forFormat(A2A_CARD);
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).doesNotContain("\"goals\"");
}

@Test
void combined_standing_and_current_goals_both_render() {
    var d = AgentDescriptor.builder()
        .agentId("a").name("n").slot("s").tenancyId("t")
        .goals(List.of(new AgentGoal("find-diamond", "Find it", GoalPriority.PRIMARY, Visibility.PUBLIC)))
        .build();
    var ctx = AgentPromptContext.forFormat(MARKDOWN)
        .withGoal(GoalContext.of("Search room 3 for clues"));
    var result = pipeline.assemble(pipeline.buildStage1(d, ctx),
        Optional.empty(), Optional.empty(), d, ctx);
    assertThat(result.content()).contains("## Objectives");
    assertThat(result.content()).contains("Find it");
    assertThat(result.content()).contains("## Current Goal");
    assertThat(result.content()).contains("Search room 3");
}

@Test
void descriptor_payload_includes_goals_for_cache_key() {
    var d = descriptorWithGoalsAndConstraints();
    var payload = pipeline.buildDescriptorPayload(d, MARKDOWN);
    assertThat(payload.has("goals")).isTrue();
    assertThat(payload.has("constraints")).isTrue();
}
```

- [ ] **Step 2: Run tests to verify they fail**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="EidosRenderPipelineTest"`
Expected: FAIL

- [ ] **Step 3: Implement rendering pipeline changes**

In `buildDescriptorPayload()`, add after `addIfPresent(node, "briefing", descriptor.briefing())`:

```java
if (descriptor.goals() != null && !descriptor.goals().isEmpty()) {
    final List<AgentGoal> goalsToInclude = format == RenderFormat.A2A_CARD
        ? descriptor.publicGoals() : descriptor.goals();
    if (!goalsToInclude.isEmpty()) {
        final ArrayNode goalsArray = node.putArray("goals");
        for (final AgentGoal goal : goalsToInclude) {
            final ObjectNode goalNode = goalsArray.addObject();
            goalNode.put("name", goal.name());
            goalNode.put("description", goal.description());
            goalNode.put("priority", goal.priority().name());
            goalNode.put("visibility", goal.visibility().name());
        }
    }
}

if (descriptor.constraints() != null && !descriptor.constraints().isEmpty()) {
    final List<AgentConstraint> constraintsToInclude = format == RenderFormat.A2A_CARD
        ? descriptor.publicConstraints() : descriptor.constraints();
    if (!constraintsToInclude.isEmpty()) {
        final ArrayNode constraintsArray = node.putArray("constraints");
        for (final AgentConstraint c : constraintsToInclude) {
            final ObjectNode cNode = constraintsArray.addObject();
            cNode.put("name", c.name());
            cNode.put("description", c.description());
            cNode.put("visibility", c.visibility().name());
        }
    }
}
```

Add two new assembly methods:

```java
private void assembleMarkdownObjectives(final StringBuilder sb, final AgentDescriptor descriptor) {
    if (descriptor.goals().isEmpty()) return;
    sb.append("\n## Objectives\n");
    descriptor.goals().stream()
        .sorted(Comparator.comparing(AgentGoal::priority)
            .thenComparing(AgentGoal::name))
        .forEach(g -> sb.append("- **[").append(g.priority().name()).append("]** ")
            .append(g.description()).append("\n"));
}

private void assembleMarkdownConstraints(final StringBuilder sb, final AgentDescriptor descriptor) {
    if (descriptor.constraints().isEmpty()) return;
    sb.append("\n## Constraints\n");
    descriptor.constraints().stream()
        .sorted(Comparator.comparing(AgentConstraint::name))
        .forEach(c -> sb.append("- ").append(c.description()).append("\n"));
}
```

Add import for `java.util.Comparator` if not present.

In `assembleMarkdown()`, add these calls after `assembleMarkdownCapabilities(sb, descriptor)` and before `if (enrichment.isPresent()...` (disposition):

```java
assembleMarkdownObjectives(sb, descriptor);
assembleMarkdownConstraints(sb, descriptor);
```

In `assembleProse()`, add after the capabilities block and before the disposition block:

```java
if (!descriptor.goals().isEmpty()) {
    var sorted = descriptor.goals().stream()
        .sorted(Comparator.comparing(AgentGoal::priority).thenComparing(AgentGoal::name))
        .toList();
    var primary = sorted.stream().filter(g -> g.priority() == GoalPriority.PRIMARY).toList();
    var secondary = sorted.stream().filter(g -> g.priority() == GoalPriority.SECONDARY).toList();
    sb.append("\nPrimary objectives: ");
    sb.append(primary.stream().map(AgentGoal::description).collect(Collectors.joining("; ")));
    sb.append(".");
    if (!secondary.isEmpty()) {
        sb.append(" Also: ");
        sb.append(secondary.stream().map(AgentGoal::description).collect(Collectors.joining("; ")));
        sb.append(".");
    }
    sb.append("\n");
}

if (!descriptor.constraints().isEmpty()) {
    sb.append("\nConstraints: ");
    sb.append(descriptor.constraints().stream()
        .sorted(Comparator.comparing(AgentConstraint::name))
        .map(AgentConstraint::description).collect(Collectors.joining(". ")));
    sb.append(".\n");
}
```

In `assembleA2aCard()`, add after the capabilities block and before the `try` serialization:

```java
final List<AgentGoal> publicGoals = descriptor.publicGoals();
if (!publicGoals.isEmpty()) {
    final ArrayNode goalsArray = card.putArray("goals");
    for (final AgentGoal goal : publicGoals.stream()
            .sorted(Comparator.comparing(AgentGoal::priority).thenComparing(AgentGoal::name))
            .toList()) {
        final ObjectNode goalNode = goalsArray.addObject();
        goalNode.put("name", goal.name());
        goalNode.put("description", goal.description());
        goalNode.put("priority", goal.priority().name());
    }
}

final List<AgentConstraint> publicConstraints = descriptor.publicConstraints();
if (!publicConstraints.isEmpty()) {
    final ArrayNode constraintsArray = card.putArray("constraints");
    for (final AgentConstraint c : publicConstraints.stream()
            .sorted(Comparator.comparing(AgentConstraint::name))
            .toList()) {
        final ObjectNode cNode = constraintsArray.addObject();
        cNode.put("name", c.name());
        cNode.put("description", c.description());
    }
}
```

- [ ] **Step 4: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="EidosRenderPipelineTest"`
Expected: PASS

- [ ] **Step 5: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/renderer/EidosRenderPipeline.java \
       runtime/src/test/java/io/casehub/eidos/runtime/renderer/EidosRenderPipelineTest.java
git commit -m "feat(#100): render goals and constraints in MARKDOWN, PROSE, and A2A_CARD

Refs #100"
```

---

### Task 6: JPA Persistence

**Files:**
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentGoalEntity.java`
- Create: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentConstraintEntity.java`
- Create: `runtime/src/main/resources/db/eidos/migration/V7__goals_constraints.sql`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java`
- Modify: `runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java`
- Test: `runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java`

**Interfaces:**
- Consumes: `AgentGoal`, `AgentConstraint`, `GoalPriority`, `Visibility` from Task 1; `AgentDescriptor.goals()`, `.constraints()` from Task 2
- Produces: JPA round-trip persistence for goals and constraints

- [ ] **Step 1: Write failing JPA round-trip test**

Add to `JpaAgentRegistryTest.java`:

```java
@Test
void goals_and_constraints_persist_and_retrieve() {
    var goals = List.of(
        new AgentGoal("find-diamond", "Find it", GoalPriority.PRIMARY, Visibility.PUBLIC),
        new AgentGoal("help-others", "Help", GoalPriority.SECONDARY, Visibility.PRIVATE));
    var constraints = List.of(
        new AgentConstraint("trust-everyone", "Trust by default", Visibility.PUBLIC),
        new AgentConstraint("oblivious", "Don't notice danger", Visibility.PRIVATE));
    var d = AgentDescriptor.builder()
        .agentId("penelope").name("Penelope").slot("heroine").tenancyId("wacky-manor")
        .goals(goals).constraints(constraints).build();

    registry.register(d);
    var retrieved = registry.findById("penelope", "wacky-manor").orElseThrow();

    assertThat(retrieved.goals()).hasSize(2);
    assertThat(retrieved.goals().stream().map(AgentGoal::name))
        .containsExactlyInAnyOrder("find-diamond", "help-others");
    var findDiamond = retrieved.goals().stream()
        .filter(g -> g.name().equals("find-diamond")).findFirst().orElseThrow();
    assertThat(findDiamond.priority()).isEqualTo(GoalPriority.PRIMARY);
    assertThat(findDiamond.visibility()).isEqualTo(Visibility.PUBLIC);

    assertThat(retrieved.constraints()).hasSize(2);
    var trust = retrieved.constraints().stream()
        .filter(c -> c.name().equals("trust-everyone")).findFirst().orElseThrow();
    assertThat(trust.visibility()).isEqualTo(Visibility.PUBLIC);
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime -Dtest="JpaAgentRegistryTest#goals_and_constraints_persist_and_retrieve"`
Expected: FAIL — mapper/entity don't handle goals/constraints

- [ ] **Step 3: Create Flyway migration**

Create `runtime/src/main/resources/db/eidos/migration/V7__goals_constraints.sql`:

```sql
CREATE TABLE agent_goal (
    id             BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    descriptor_id  BIGINT       NOT NULL REFERENCES agent_descriptor(internal_id) ON DELETE CASCADE,
    agent_id       VARCHAR(255) NOT NULL,
    tenancy_id     VARCHAR(255) NOT NULL,
    name           VARCHAR(100) NOT NULL,
    description    TEXT         NOT NULL,
    priority       VARCHAR(20)  NOT NULL,
    visibility     VARCHAR(20)  NOT NULL,
    UNIQUE (descriptor_id, name)
);

CREATE TABLE agent_constraint (
    id             BIGINT GENERATED BY DEFAULT AS IDENTITY PRIMARY KEY,
    descriptor_id  BIGINT       NOT NULL REFERENCES agent_descriptor(internal_id) ON DELETE CASCADE,
    agent_id       VARCHAR(255) NOT NULL,
    tenancy_id     VARCHAR(255) NOT NULL,
    name           VARCHAR(100) NOT NULL,
    description    TEXT         NOT NULL,
    visibility     VARCHAR(20)  NOT NULL,
    UNIQUE (descriptor_id, name)
);
```

- [ ] **Step 4: Create entity classes**

Create `AgentGoalEntity.java`:

```java
package io.casehub.eidos.runtime.registry.jpa;

import jakarta.persistence.*;

@Entity
@Table(name = "agent_goal",
       uniqueConstraints = @UniqueConstraint(columnNames = {"descriptor_id", "name"}))
public class AgentGoalEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "descriptor_id", nullable = false)
    AgentDescriptorEntity descriptor;

    @Column(name = "agent_id")   String agentId;
    @Column(name = "tenancy_id") String tenancyId;

    @Column(nullable = false) String name;
    @Column(columnDefinition = "TEXT", nullable = false) String description;
    @Column(nullable = false) String priority;
    @Column(nullable = false) String visibility;
}
```

Create `AgentConstraintEntity.java`:

```java
package io.casehub.eidos.runtime.registry.jpa;

import jakarta.persistence.*;

@Entity
@Table(name = "agent_constraint",
       uniqueConstraints = @UniqueConstraint(columnNames = {"descriptor_id", "name"}))
public class AgentConstraintEntity {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    Long id;

    @ManyToOne(fetch = FetchType.LAZY)
    @JoinColumn(name = "descriptor_id", nullable = false)
    AgentDescriptorEntity descriptor;

    @Column(name = "agent_id")   String agentId;
    @Column(name = "tenancy_id") String tenancyId;

    @Column(nullable = false) String name;
    @Column(columnDefinition = "TEXT", nullable = false) String description;
    @Column(nullable = false) String visibility;
}
```

- [ ] **Step 5: Add collections to AgentDescriptorEntity**

Add to `AgentDescriptorEntity`:

```java
@OneToMany(mappedBy = "descriptor", cascade = CascadeType.ALL,
           fetch = FetchType.LAZY, orphanRemoval = true)
List<AgentGoalEntity> goals = new ArrayList<>();

@OneToMany(mappedBy = "descriptor", cascade = CascadeType.ALL,
           fetch = FetchType.LAZY, orphanRemoval = true)
List<AgentConstraintEntity> constraints = new ArrayList<>();
```

- [ ] **Step 6: Update AgentDescriptorMapper**

In `toRecord()`, add goals and constraints to the constructor call. The positional constructor now has 20 arguments — add goals and constraints as the last two:

```java
AgentDescriptor toRecord(AgentDescriptorEntity e) {
    return new AgentDescriptor(
        e.agentId, e.name, e.version, e.provider,
        e.modelFamily, e.modelVersion, e.weightsFingerprint,
        e.domainVocabulary, e.slotVocabulary, e.dispositionVocabulary,
        readJson(e.axisVocabularies, new TypeReference<Map<DispositionAxis, String>>() {}),
        e.slot,
        e.capabilities.stream().map(this::toCapability).toList(),
        readJson(e.disposition, AgentDisposition.class),
        e.jurisdiction, e.dataHandlingPolicy, e.tenancyId,
        e.briefing,
        e.goals.stream().map(this::toGoal).toList(),
        e.constraints.stream().map(this::toConstraint).toList()
    );
}
```

In `toEntity()`, add before `return e`:

```java
d.goals().stream()
    .map(g -> toGoalEntity(g, e))
    .forEach(e.goals::add);
d.constraints().stream()
    .map(c -> toConstraintEntity(c, e))
    .forEach(e.constraints::add);
```

Add new mapping methods:

```java
private AgentGoal toGoal(AgentGoalEntity g) {
    return new AgentGoal(g.name, g.description,
        GoalPriority.valueOf(g.priority),
        Visibility.valueOf(g.visibility));
}

private AgentGoalEntity toGoalEntity(AgentGoal g, AgentDescriptorEntity parent) {
    var e = new AgentGoalEntity();
    e.descriptor = parent;
    e.agentId    = parent.agentId;
    e.tenancyId  = parent.tenancyId;
    e.name       = g.name();
    e.description = g.description();
    e.priority   = g.priority().name();
    e.visibility = g.visibility().name();
    return e;
}

private AgentConstraint toConstraint(AgentConstraintEntity c) {
    return new AgentConstraint(c.name, c.description,
        Visibility.valueOf(c.visibility));
}

private AgentConstraintEntity toConstraintEntity(AgentConstraint c, AgentDescriptorEntity parent) {
    var e = new AgentConstraintEntity();
    e.descriptor = parent;
    e.agentId    = parent.agentId;
    e.tenancyId  = parent.tenancyId;
    e.name       = c.name();
    e.description = c.description();
    e.visibility = c.visibility().name();
    return e;
}
```

- [ ] **Step 7: Run tests**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn test -pl runtime`
Expected: PASS

- [ ] **Step 8: Run full project build**

Run: `JAVA_HOME=$(/usr/libexec/java_home -v 26) mvn clean install`
Expected: PASS — all modules green

- [ ] **Step 9: Commit**

```bash
git add runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentGoalEntity.java \
       runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentConstraintEntity.java \
       runtime/src/main/resources/db/eidos/migration/V7__goals_constraints.sql \
       runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorEntity.java \
       runtime/src/main/java/io/casehub/eidos/runtime/registry/jpa/AgentDescriptorMapper.java \
       runtime/src/test/java/io/casehub/eidos/runtime/registry/JpaAgentRegistryTest.java
git commit -m "feat(#100): JPA persistence for goals and constraints (Flyway V7)

Refs #100"
```
