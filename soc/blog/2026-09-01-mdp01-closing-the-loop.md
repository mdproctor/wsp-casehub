---
layout: post
title: "Closing the loop"
date: 2026-09-01
entry_type: note
subtype: diary
projects: [casehubio/soc]
tags: [soc, cbr, case-based-reasoning, retrieval, triage]
series: issue-35-cbr-retrieval
---

# Closing the loop

The SOC app had a binding called `cbr-retrieval` that fired first in every investigation — before IOC enrichment, before ATT&CK mapping, before containment. Its job: find similar past incidents and feed them to every downstream worker so triage starts with context, not from scratch. The capability was defined, the retrieve service was built, the retain service was wired into the case outcome lifecycle. Everything except the one thing that mattered: a worker to call it.

Every investigation FAULTed silently on `cbr-retrieval` — "no eligible workers for capability" — and the engine moved on. The pipeline worked, but every case started cold. Past resolutions existed in memory and were never consulted.

The fix was five files of wiring. `RuleCbrRetrievalWorker` wraps `SocCbrRetrieveService` as a `Worker`, registered at position zero in the descriptor so it runs first. `SocCaseHub` injects the retrieve service via CDI and passes it through the descriptor constructor — the same pattern used for the `ChatModel` LLM dependency. No new abstractions, no new SPIs.

The seed data loader populates five representative incidents at startup — credential harvesting, brute force, malware, phishing, lateral movement — each with realistic ATT&CK techniques, IOC types, and resolution outcomes. A parameterised test verifies that querying with a similar alert retrieves the matching seed incident first.

What makes this satisfying is the asymmetry between effort and effect. The CBR infrastructure was already comprehensive — case type, schema registration, feature extraction, retain-on-outcome, REST endpoint, similarity queries. Connecting the last wire turned a silently FAULTing capability into working institutional memory. The next credential-harvesting alert will see how the last one was resolved.

The tenantId question is still a placeholder — `DEFAULT_TENANT` constant for now because `WorkerScope` doesn't expose tenant context. When multi-tenancy is production-ready, that's a one-line change. Not worth building machinery around today.
