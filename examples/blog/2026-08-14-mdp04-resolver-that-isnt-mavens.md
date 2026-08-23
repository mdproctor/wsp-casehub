---
layout: post
title: "The Resolver That Isn't Maven's"
date: 2026-08-14
entry_type: note
subtype: diary
projects: [casehub-examples]
tags: [quarkus, maven, debugging, infrastructure]
series: issue-41-autonomous-agent-template
---

*Continues from [Goals Think, Plans Act](2026-08-14-mdp03-goals-think-plans-act.md).*

Resumed slot 107 to verify #44 — plan structure. Five commits had already landed across previous sessions: data model, formation strategy, revision strategy, evaluator wiring, and LLM eval tests. The implementation was done. I just needed to confirm tests passed.

They didn't. `ManorResourceProfileTest` failed during Quarkus test discovery with `ClassSelector resolution failed`. The error looked like a classloading problem. It wasn't.

The real failure was buried in the Quarkus bootstrap log: `Failed to resolve artifact io.casehub:casehub-eidos-eval:jar:0.2-SNAPSHOT`. The jar existed in `~/.m2/repository`. `mvn compile` and `mvn test-compile` both succeeded. `mvn dependency:resolve` succeeded. Only the Quarkus test runtime rejected it.

I spent time on the wrong trail. Deleted `_remote.repositories`. Deleted `resolver-status.properties`. Tried offline mode. Reinstalled eidos. Nothing worked. The root cause: Quarkus's `ApplicationDependencyResolver` is not Maven's resolver. It has its own dependency resolution path, and it requires artifacts to be resolvable from a *named* repository source. The default `~/.m2/settings.xml` declared a `github` remote for GitHub Packages. The artifact was never published there. The Quarkus resolver checked, found nothing, and rejected the locally-installed jar as "present, but unavailable."

The fix was in front of me the whole time. The slot has a `slot-settings.xml` that defines `host-m2` — a file:// URL pointing at `~/.m2/repository`. Using `-s slot-settings.xml` replaces the default settings entirely. The `github` remote disappears. The Quarkus resolver only sees `host-m2`, finds the artifact, and moves on. All 50 test classes pass.

Updated CLAUDE.md to include `-s slot-settings.xml` in every Maven command. Marked #44 complete — the plan structure work (goals → plans → thinking, three-layer cognitive model) is done. Queue advances to #45: trust and personality.

The Quarkus resolver divergence is the kind of thing that costs hours on the second encounter because the error message points somewhere else entirely. Submitted a fix to the garden entry that had previously listed this as unsolved.
