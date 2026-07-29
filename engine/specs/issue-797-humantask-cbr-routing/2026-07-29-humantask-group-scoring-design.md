# HumanTask Group Scoring via Group Membership Resolution

**Issue:** engine#757
**Date:** 2026-07-29
**Status:** Draft

## Problem

`HumanTaskRoutingStrategy` scores individual `candidateUsers` but cannot score
`candidateGroups` because group membership resolution is outside the engine's scope.
The `ContextConstraint.Prefer` and `ContextConstraint.Exclude` effects carry `groups`
fields but the constraint strategy ignores them at runtime. CBR scoring only operates
on directly-nominated users.

## Approach

Use the existing `GroupMembershipProvider` SPI from `casehub-platform-api`
(`io.casehub.platform.api.identity`). It provides `membersOf(groupName, tenancyId)
→ Set<GroupMember>` and is already used by `TemplateExpander` in the work repo for
the same purpose. `casehub-platform-api` is already on the engine's dependency tree
(`api/pom.xml`, `runtime/pom.xml`). No new SPI required.

## Design

### 1. Handler expands groups before strategy dispatch

`CaseContextChangedEventHandler.publishHumanTaskSchedule()` injects
`GroupMembershipProvider` and expands all candidate groups at routing time:

```
for each group in resolvedGroups:
    members = groupMembershipProvider.membersOf(group, tenancyId)
    groupMembership[group] = members.map(GroupMember::actorId)
```

The handler always expands. `MockGroupMembershipProvider` (`@DefaultBean`) returns
empty — zero cost when no real provider is configured.

### 2. HumanTaskCandidates gains groupMembership

```java
public record HumanTaskCandidates(
    Set<String> groups,
    Set<String> users,
    Map<String, Set<String>> groupMembership) {

    // Compact constructor: null-safe, defensive copies (deep for map)

    public Set<String> allUsers() {
        // union of direct users + all group member actor IDs
    }
}
```

Backward-compatible factory for tests:
```java
public static HumanTaskCandidates of(Set<String> groups, Set<String> users) {
    return new HumanTaskCandidates(groups, users, Map.of());
}
```

### 3. CBR strategy scores allUsers()

`CbrHumanTaskRoutingStrategy.select()` changes:
- Guard: `candidates.allUsers().isEmpty()` (was `candidates.users().isEmpty()`)
- Passes `candidates.allUsers()` as `eligibleWorkerIds` to
  `ExperienceAnalyser.workerSuccessRates()`
- Result: `Enriched(candidates.groups(), candidates.users(), scores)`
  — groups and direct users pass through unchanged; scores include
  group-expanded users

### 4. Constraint strategy applies group effects

`ConstraintHumanTaskRoutingStrategy.select()` changes:

**Exclude(groups):**
- Resolve group members from `candidates.groupMembership()`
- Remove members from `eligibleUsers`
- Remove group from `eligibleGroups` (new mutable set, initialized from
  `candidates.groups()`) — otherwise the work repo would override the
  engine's exclusion decision

**Prefer(groups):**
- Resolve group members from `candidates.groupMembership()`
- For each member in `eligibleUsers`, merge `constraint.weight()` into
  their score (same logic as user-level Prefer)
- Groups stay in `eligibleGroups` (preference doesn't restrict visibility)

**Workload constraint:**
- Operate on `allUsers()` union — group-expanded users participate in
  workload checks and load-balance scoring

**Eligible users initialization:**
- Start from `candidates.allUsers()` instead of `candidates.users()`

### 5. Unchanged components

- **HumanTaskScheduleEvent** — no schema change. Scores for group-expanded
  users flow through the existing `candidateScores` map. `resolvedCandidateUsers`
  stays as original direct users. `resolvedCandidateGroups` may have groups
  removed by constraint exclusion.
- **HumanTaskRoutingResult.Enriched** — same three fields. `candidateScores`
  keys are individual actor IDs (direct or group-expanded), never group names.
  Javadoc updated to remove the engine#757 caveat.
- **HumanTaskRoutingContext** — no new fields. Membership is about the
  candidates, not the case context.
- **NoOpHumanTaskRoutingStrategy** — returns `Unchanged`, unaffected.
- **ExperienceAnalyser** — no changes. Already accepts any `Set<String>`
  as eligible IDs.

### 6. Semantic decisions

| Decision | Rationale |
|----------|-----------|
| No group-level scores in `candidateScores` | Work repo assigns to individuals. Group aggregation (avg/max) is a policy decision for the consumer. |
| `Enriched.candidateUsers()` = original direct users | Work repo resolves groups independently for assignment. Scores for group-expanded users are matched by actor ID after work repo's own resolution. |
| `Enriched.candidateGroups()` CAN be modified | Constraint Exclude removes groups — otherwise exclusion is meaningless (work repo would still show the task). |
| Handler always expands | Mock returns empty (zero cost). Real provider implies real strategy. No lazy path needed. |
| Membership on candidates, not context | It's data about the candidate groups, not about the case/task. `allUsers()` as a derived method on candidates is natural. |

## Testing

- **HumanTaskCandidatesTest** — `allUsers()` union, `groupMembership` defensive copy,
  null-safe construction, backward-compatible `of()` factory
- **CbrHumanTaskRoutingStrategyTest** — scoring includes group-expanded users; empty
  allUsers returns Unchanged; overlap between direct user and group member produces
  single score
- **ConstraintHumanTaskRoutingStrategyTest** — Exclude(groups) removes group from
  result and members from eligible; Prefer(groups) boosts member scores; workload
  constraint applies to allUsers; all-excluded-via-groups escalates
- **CaseContextChangedEventHandler** — verify `GroupMembershipProvider.membersOf()`
  called per group; verify expanded membership threaded to candidates; verify
  Enriched result with group-excluded groups flows to HumanTaskScheduleEvent

## Retention impact

`CbrCaseRetainObserver.buildRoutingKeyMap()` already includes `HumanTaskTarget`
bindings with null `capabilityName`. Plan traces for humanTask bindings carry
`workerName` which is the individual actor who completed the task — group-expanded
users who complete tasks are naturally captured in the CBR case base. No retention
changes needed.
