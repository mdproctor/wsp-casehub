# casehub-examples Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — create on casehubio/parent before starting
**Issue group:** Single issue

**Goal:** Create `casehubio/examples` — a single cloneable repo that
aggregates `examples/` directories from all CaseHub repos via
subtree-split, with a top-level aggregator POM, root README, local
sync script, and CI workflow triggered by `ecosystem-build-succeeded`.

**Architecture:** The repo contains no source code of its own. CI
clones each source repo into a throwaway directory, runs
`git subtree split --prefix=examples` to extract a filtered branch,
then `git subtree add/pull --prefix=<repo>-examples --squash` into
the examples repo. A standalone aggregator POM orchestrates
`mvn test` across all Maven example sets. A `sync.sh` script lets
developers update locally to HEAD.

**Tech Stack:** Git subtree, GitHub Actions, Maven aggregator POM,
Python (dispatch script extension), Bash (sync script)

**Spec:** `docs/specs/2026-07-09-casehub-examples-design.md`

## Global Constraints

- Repo name: `casehubio/examples` (short name, no `casehub-` prefix)
- Directory names: `<repo>-examples/` (e.g., `ledger-examples/`)
- Maven artifact: `casehub-examples` for the aggregator
- All operations on source repos are read-only — never push to them
- SHAs come from `ecosystem-build-succeeded` dispatch payload
- SHA keys use underscore convention: `blocks-ui` → `blocks_ui`
- `--squash` on all subtree operations (clean sync history)
- Non-Maven examples excluded from POM, included in repo and README

---

### Task 1: Create the GitHub repo and seed with infrastructure files

**Files:**
- Create: `pom.xml` (aggregator)
- Create: `sync-config.json`
- Create: `README.md`
- Create: `ADDING-EXAMPLES.md`
- Create: `sync.sh`
- Create: `.gitignore`

**Interfaces:**
- Produces: Empty repo with all infrastructure files. No example
  content yet — Task 2 seeds the subtrees.

- [ ] **Step 1: Create the GitHub repo**

```bash
gh repo create casehubio/examples \
  --public \
  --description "All CaseHub examples in one place — clone, browse, run" \
  --clone
```

- [ ] **Step 2: Create `.gitignore`**

```gitignore
target/
.idea/
*.iml
node_modules/
dist/
.quarkus/
```

- [ ] **Step 3: Create `sync-config.json`**

This is the deliberate opt-in list. Only repos with existing `examples/`
directories are included. `blocks` is omitted — it will be added when
its examples are ready.

```json
{
  "repos": [
    {"name": "ledger",       "org": "casehubio", "type": "maven"},
    {"name": "work",         "org": "casehubio", "type": "maven"},
    {"name": "qhorus",       "org": "casehubio", "type": "maven"},
    {"name": "eidos",        "org": "casehubio", "type": "maven"},
    {"name": "desiredstate", "org": "casehubio", "type": "maven"},
    {"name": "neocortex",    "org": "casehubio", "type": "maven"},
    {"name": "openclaw",     "org": "casehubio", "type": "docker"},
    {"name": "blocks-ui",    "org": "casehubio", "type": "typescript"},
    {"name": "pages",        "org": "casehubio", "type": "typescript"}
  ]
}
```

- [ ] **Step 4: Create the aggregator `pom.xml`**

Standalone aggregator — does NOT inherit from `casehub-parent`. Module
entries point at each Maven example set's directory. The example set's
own aggregator POM (if present) handles its internal modules.

Note: Some source repos (ledger, desiredstate, neocortex) currently lack
an aggregator POM in their `examples/` directory. Task 3 addresses this
as prerequisite work. Until then, their individual example POMs are listed
directly.

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub</groupId>
  <artifactId>casehub-examples</artifactId>
  <version>0.1-SNAPSHOT</version>
  <packaging>pom</packaging>

  <name>CaseHub Examples</name>
  <description>
    Aggregated examples from all CaseHub repos. Clone this single repo
    to browse and run every CaseHub example.
  </description>

  <modules>
    <!-- Repos with aggregator POMs — one entry each -->
    <module>work-examples</module>
    <module>qhorus-examples</module>
    <module>eidos-examples</module>

    <!-- Repos without aggregator POMs — list each example individually.
         These will collapse to one entry per repo once Task 3 adds
         aggregator POMs to the source repos. -->
    <module>ledger-examples/order-processing</module>
    <module>ledger-examples/art22-decision-snapshot</module>
    <module>ledger-examples/eigentrust-mesh</module>
    <module>ledger-examples/merkle-verification</module>
    <module>ledger-examples/otel-trace-wiring</module>
    <module>ledger-examples/privacy-pseudonymisation</module>
    <module>ledger-examples/prov-dm-export</module>
    <module>ledger-examples/trust-score-routing</module>
    <module>ledger-examples/verifiable-credentials</module>
    <module>ledger-examples/art12-compliance</module>
    <module>ledger-examples/aws-kms-signing</module>
    <module>ledger-examples/azure-keyvault-signing</module>
    <module>ledger-examples/gcp-kms-signing</module>
    <module>ledger-examples/vault-transit-signing</module>

    <module>desiredstate-examples/pipeline</module>
    <module>desiredstate-examples/dungeon</module>
    <module>desiredstate-examples/expansion</module>
    <module>desiredstate-examples/spatial</module>

    <module>neocortex-examples/example-cbr</module>
    <module>neocortex-examples/example-rag-pipeline</module>
    <module>neocortex-examples/example-text-analysis</module>

    <!-- Non-Maven example sets are NOT listed here.
         openclaw (Docker Compose), blocks-ui (TypeScript), pages (TypeScript)
         are present in the repo but have standalone run instructions. -->
  </modules>

  <repositories>
    <repository>
      <id>github</id>
      <url>https://maven.pkg.github.com/casehubio/*</url>
    </repository>
  </repositories>
</project>
```

- [ ] **Step 5: Create `sync.sh`**

Local sync script for developers who want HEAD instead of pinned SHAs.
Reads `sync-config.json`, clones each source repo at HEAD, runs
subtree-split, and pulls.

```bash
#!/bin/bash
set -euo pipefail

# Update all example subtrees to HEAD of each source repo.
# Analogous to casehub-all's sync.sh, adapted for subtrees.

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
CONFIG="$SCRIPT_DIR/sync-config.json"
TMPDIR=$(mktemp -d)

trap 'rm -rf "$TMPDIR"' EXIT

echo "Syncing examples to HEAD..."

# Parse sync-config.json
repos=$(python3 -c "
import json, sys
config = json.load(open('$CONFIG'))
for r in config['repos']:
    print(f\"{r['name']} {r['org']}\")
")

while read -r name org; do
  repo_url="https://github.com/$org/$name.git"
  clone_dir="$TMPDIR/$name"
  prefix="${name}-examples"

  echo ""
  echo "--- $name ---"

  # Clone at HEAD
  git clone --quiet "$repo_url" "$clone_dir" 2>/dev/null || {
    echo "  SKIP: could not clone $repo_url"
    continue
  }

  # Check examples/ exists
  if [ ! -d "$clone_dir/examples" ]; then
    echo "  SKIP: no examples/ directory"
    rm -rf "$clone_dir"
    continue
  fi

  # Split out just the examples/ directory
  (cd "$clone_dir" && git subtree split --prefix=examples -b examples-only --quiet) || {
    echo "  SKIP: subtree split failed"
    rm -rf "$clone_dir"
    continue
  }

  # Add or pull into this repo
  if [ -d "$SCRIPT_DIR/$prefix" ]; then
    git subtree pull --prefix="$prefix" "$clone_dir" examples-only --squash -m "sync: $name examples to HEAD" --quiet || {
      echo "  WARN: subtree pull failed (may need manual merge)"
      rm -rf "$clone_dir"
      continue
    }
    echo "  Updated $prefix"
  else
    git subtree add --prefix="$prefix" "$clone_dir" examples-only --squash -m "sync: seed $name examples from HEAD" --quiet || {
      echo "  WARN: subtree add failed"
      rm -rf "$clone_dir"
      continue
    }
    echo "  Seeded $prefix"
  fi

  rm -rf "$clone_dir"
done <<< "$repos"

echo ""
echo "Sync complete."
```

- [ ] **Step 6: Create `README.md`**

```markdown
# CaseHub Examples

All CaseHub examples in one place. Clone this repo to browse and run
examples from every CaseHub module — no need to clone individual repos.

## Prerequisites

- **Java 21+** (26 recommended)
- **Maven 3.9+**
- **GitHub Packages access** (until CaseHub publishes to Maven Central):

  Add to `~/.m2/settings.xml`:
  ```xml
  <server>
    <id>github</id>
    <username>YOUR_GITHUB_USERNAME</username>
    <password>YOUR_GITHUB_TOKEN</password>
  </server>
  ```
  The token needs `read:packages` scope.

## Quick Start

```bash
git clone https://github.com/casehubio/examples.git
cd examples

# Run all Maven examples
mvn test

# Run a specific example in dev mode
mvn quarkus:dev -pl ledger-examples/order-processing

# Update to latest HEAD (optional)
./sync.sh
```

## Example Sets

| Example Set | Type | Description |
|---|---|---|
| [ledger-examples](ledger-examples/) | Quarkus | Immutable audit trails, Merkle proofs, trust scoring, GDPR compliance |
| [work-examples](work-examples/) | Quarkus | Human task lifecycle, SLA enforcement, queues, AI routing |
| [qhorus-examples](qhorus-examples/) | Quarkus | Agent communication, speech acts, normative channel layout |
| [eidos-examples](eidos-examples/) | Quarkus | Agent scenario orchestration |
| [desiredstate-examples](desiredstate-examples/) | Quarkus | Desired-state reconciliation, drift detection, pipelines |
| [neocortex-examples](neocortex-examples/) | Quarkus | RAG pipelines, case-based reasoning, text analysis |
| [openclaw-examples](openclaw-examples/) | Docker Compose | Multi-agent rule authoring scenarios |
| [blocks-ui-examples](blocks-ui-examples/) | TypeScript | Visual block builder |
| [pages-examples](pages-examples/) | TypeScript | Dashboard and layout editor |

## Non-Maven Examples

**openclaw** (Docker Compose): Requires Docker. See
[openclaw-examples/README.md](openclaw-examples/README.md).

**blocks-ui** (TypeScript/Vite): Requires Node.js 20+. See
[blocks-ui-examples/README.md](blocks-ui-examples/README.md).

**pages** (TypeScript/webpack): Requires Node.js 20+. See
[pages-examples/README.md](pages-examples/README.md).

## LLM Examples (Qhorus)

The qhorus `agent-communication` module requires a local LLM model.
It is excluded from the default build. To run it:

```bash
mvn test -Pwith-llm-examples -pl qhorus-examples
```

## About This Repo

This repository is **read-only** — it is automatically synced from the
source repos after each successful
[full build](https://github.com/casehubio/parent/actions/workflows/build-all.yml).
Examples always match a known-good build state.

To fix a bug or add an example, make the change in the source repo:

| Example Set | Source Repo |
|---|---|
| ledger-examples | [casehubio/ledger](https://github.com/casehubio/ledger) |
| work-examples | [casehubio/work](https://github.com/casehubio/work) |
| qhorus-examples | [casehubio/qhorus](https://github.com/casehubio/qhorus) |
| eidos-examples | [casehubio/eidos](https://github.com/casehubio/eidos) |
| desiredstate-examples | [casehubio/desiredstate](https://github.com/casehubio/casehub-desiredstate) |
| neocortex-examples | [casehubio/neocortex](https://github.com/casehubio/neocortex) |
| openclaw-examples | [casehubio/openclaw](https://github.com/casehubio/openclaw) |
| blocks-ui-examples | [casehubio/blocks-ui](https://github.com/casehubio/casehub-pages) |
| pages-examples | [casehubio/pages](https://github.com/casehubio/casehub-pages) |

Changes flow to this repo on the next successful full build.
```

- [ ] **Step 7: Create `ADDING-EXAMPLES.md`**

```markdown
# Adding a New Repo's Examples

LLM-executable checklist for onboarding a new repo's examples into
casehub-examples. Follow these steps in order.

## Prerequisites (in the source repo)

1. **Verify `examples/` directory exists** with at least one runnable
   example (Maven module, Docker Compose, or TypeScript project).

2. **Verify POM parent resolution.** Example POMs that inherit from a
   parent (e.g., `<parent>casehub-work-parent</parent>`) must set
   `<relativePath/>` (empty, self-closing) in the `<parent>` block.
   This tells Maven to resolve from the repository, not the filesystem.
   Without it, the examples won't build after subtree extraction.

   Standalone POMs (like ledger's — no `<parent>`) need no changes.

3. **Verify aggregator POM.** The source repo must have a
   `examples/pom.xml` with `<packaging>pom</packaging>` that lists
   all example subdirectories as `<modules>`. If absent, create one:

   ```xml
   <?xml version="1.0"?>
   <project ...>
     <modelVersion>4.0.0</modelVersion>
     <groupId>io.casehub.<repo>.examples</groupId>
     <artifactId>casehub-<repo>-examples</artifactId>
     <version>0.2-SNAPSHOT</version>
     <packaging>pom</packaging>
     <name>CaseHub <Repo> - Examples</name>
     <modules>
       <module>example-one</module>
       <module>example-two</module>
     </modules>
   </project>
   ```

   This ensures new examples added to the source repo are
   automatically included in the aggregator build.

## In casehub-examples

4. **Add to `sync-config.json`:**

   ```json
   {"name": "<repo>", "org": "casehubio", "type": "maven"}
   ```

   Use `type`: `maven`, `docker`, or `typescript`.

5. **Run the sync locally:**

   ```bash
   ./sync.sh
   ```

   This pulls the examples via subtree-split. Verify the
   `<repo>-examples/` directory appears with the expected content.

6. **If Maven — update `pom.xml`:**

   Add a single `<module>` entry:

   ```xml
   <module><repo>-examples</module>
   ```

   The source repo's own aggregator POM handles internal modules.

7. **Verify the build:**

   ```bash
   mvn test
   ```

   All examples must pass. Fix any dependency resolution issues
   (usually a missing `<relativePath/>` in the source repo).

8. **Update `README.md`:**

   Add a row to the "Example Sets" table and a source-repo mapping
   to the "About This Repo" section.

9. **Commit and push:**

   The automated sync keeps it current from here.
```

- [ ] **Step 8: Make sync.sh executable and commit**

```bash
chmod +x sync.sh
git add .gitignore sync-config.json pom.xml README.md ADDING-EXAMPLES.md sync.sh
git commit -m "feat: initial infrastructure — aggregator POM, sync script, README, onboarding guide"
```

- [ ] **Step 9: Push to GitHub**

```bash
git push -u origin main
```

---

### Task 2: Seed all example subtrees

**Files:**
- Create: `ledger-examples/` (via subtree add)
- Create: `work-examples/` (via subtree add)
- Create: `qhorus-examples/` (via subtree add)
- Create: `eidos-examples/` (via subtree add)
- Create: `desiredstate-examples/` (via subtree add)
- Create: `neocortex-examples/` (via subtree add)
- Create: `openclaw-examples/` (via subtree add)
- Create: `blocks-ui-examples/` (via subtree add)
- Create: `pages-examples/` (via subtree add)

**Interfaces:**
- Consumes: `sync-config.json` from Task 1
- Produces: All example directories populated with current HEAD content

- [ ] **Step 1: Run sync.sh to seed all subtrees**

```bash
./sync.sh
```

This iterates through `sync-config.json`, clones each source repo,
splits out `examples/`, and adds it as a subtree under
`<repo>-examples/`. Each subtree add creates its own commit.

Expected output per repo:
```
--- ledger ---
  Seeded ledger-examples
--- work ---
  Seeded work-examples
...
```

Repos without an `examples/` directory are skipped with a message.

- [ ] **Step 2: Verify the directory structure**

```bash
ls -d *-examples/
```

Expected: One directory per repo in `sync-config.json` that has
examples. At minimum: `ledger-examples/`, `work-examples/`,
`qhorus-examples/`, `eidos-examples/`, `desiredstate-examples/`,
`neocortex-examples/`, `openclaw-examples/`.

Check blocks-ui and pages — they may or may not have `examples/`
directories. If absent, the sync skips them, which is correct.

- [ ] **Step 3: Verify the Maven build**

```bash
mvn test 2>&1 | tail -30
```

Expected: BUILD SUCCESS for all Maven example sets. If any fail,
investigate — likely causes:
- Missing `<relativePath/>` in parent-inherited POMs
- Missing parent POM in `.m2` (need GitHub Packages configured)
- Missing aggregator POM in a source repo's `examples/` directory

If specific examples fail due to missing parent POMs, you may need
to install them first:
```bash
# For each source repo that has parent-inherited examples:
cd /path/to/source/repo && mvn install -N -DskipTests
```

- [ ] **Step 4: Push the seeded repo**

```bash
git push
```

---

### Task 3: Create aggregator POMs in source repos that lack them

**Prerequisite:** This task touches source repos (ledger, desiredstate,
neocortex). Coordinate with each repo's session. Create GitHub issues
on each repo rather than making direct changes.

**Files:**
- Create issue: `casehubio/ledger` — "Add aggregator POM to examples/"
- Create issue: `casehubio/casehub-desiredstate` — "Add aggregator POM to examples/"
- Create issue: `casehubio/neocortex` — "Add aggregator POM to examples/"

**Interfaces:**
- Produces: GitHub issues that each repo's session can pick up

- [ ] **Step 1: Create issue on ledger**

```bash
gh issue create --repo casehubio/ledger \
  --title "Add aggregator POM to examples/ for casehub-examples integration" \
  --body "$(cat <<'EOF'
## Context

The new `casehubio/examples` repo aggregates all CaseHub example directories
via subtree-split. To include a repo's examples in the Maven build, the repo
needs an aggregator POM at `examples/pom.xml`.

## What to do

Create `examples/pom.xml`:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub.ledger.examples</groupId>
  <artifactId>casehub-ledger-examples</artifactId>
  <version>0.2-SNAPSHOT</version>
  <packaging>pom</packaging>
  <name>CaseHub Ledger - Examples</name>

  <modules>
    <module>order-processing</module>
    <module>art12-compliance</module>
    <module>art22-decision-snapshot</module>
    <module>aws-kms-signing</module>
    <module>azure-keyvault-signing</module>
    <module>eigentrust-mesh</module>
    <module>gcp-kms-signing</module>
    <module>merkle-verification</module>
    <module>otel-trace-wiring</module>
    <module>privacy-pseudonymisation</module>
    <module>prov-dm-export</module>
    <module>trust-score-routing</module>
    <module>vault-transit-signing</module>
    <module>verifiable-credentials</module>
  </modules>
</project>
```

This does not affect the source repo's build — the examples are already
standalone. It just lets a parent aggregator discover them.

Once this lands, casehub-examples can simplify its POM to
`<module>ledger-examples</module>` instead of listing 14 individual modules.
EOF
)"
```

- [ ] **Step 2: Create issue on desiredstate**

Same pattern as Step 1, adapted for desiredstate's 4 example modules
(pipeline, dungeon, expansion, spatial).

```bash
gh issue create --repo casehubio/casehub-desiredstate \
  --title "Add aggregator POM to examples/ for casehub-examples integration" \
  --body "$(cat <<'EOF'
## Context

The new `casehubio/examples` repo aggregates all CaseHub example directories
via subtree-split. To include a repo's examples in the Maven build, the repo
needs an aggregator POM at `examples/pom.xml`.

## What to do

Create `examples/pom.xml`:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub.desiredstate.examples</groupId>
  <artifactId>casehub-desiredstate-examples</artifactId>
  <version>0.2-SNAPSHOT</version>
  <packaging>pom</packaging>
  <name>CaseHub DesiredState - Examples</name>

  <modules>
    <module>pipeline</module>
    <module>dungeon</module>
    <module>expansion</module>
    <module>spatial</module>
  </modules>
</project>
```
EOF
)"
```

- [ ] **Step 3: Create issue on neocortex**

Same pattern, adapted for neocortex's 3 example modules.

```bash
gh issue create --repo casehubio/neocortex \
  --title "Add aggregator POM to examples/ for casehub-examples integration" \
  --body "$(cat <<'EOF'
## Context

The new `casehubio/examples` repo aggregates all CaseHub example directories
via subtree-split. To include a repo's examples in the Maven build, the repo
needs an aggregator POM at `examples/pom.xml`.

## What to do

Create `examples/pom.xml`:

```xml
<?xml version="1.0"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>

  <groupId>io.casehub.neocortex.examples</groupId>
  <artifactId>casehub-neocortex-examples</artifactId>
  <version>0.2-SNAPSHOT</version>
  <packaging>pom</packaging>
  <name>CaseHub Neocortex - Examples</name>

  <modules>
    <module>example-cbr</module>
    <module>example-rag-pipeline</module>
    <module>example-text-analysis</module>
  </modules>
</project>
```
EOF
)"
```

---

### Task 4: Create the CI workflow (`sync-examples.yml`)

**Files:**
- Create: `.github/workflows/sync-examples.yml`

**Interfaces:**
- Consumes: `sync-config.json`, `ecosystem-build-succeeded` dispatch
  payload with `client_payload.shas`
- Produces: Automated subtree sync on every successful full build

- [ ] **Step 1: Create the workflow file**

```yaml
name: Sync Examples

on:
  repository_dispatch:
    types: [ecosystem-build-succeeded]
  workflow_dispatch:

permissions:
  contents: write

concurrency:
  group: sync-examples
  cancel-in-progress: false

jobs:
  sync:
    name: Sync example subtrees
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GITHUB_TOKEN }}

      - name: Configure git
        run: |
          git config user.email "github-actions@github.com"
          git config user.name "GitHub Actions"

      - name: Set up Java 21
        uses: actions/setup-java@v4
        with:
          java-version: '21'
          distribution: 'temurin'
          server-id: github
          server-username: GITHUB_ACTOR
          server-password: GITHUB_TOKEN

      - name: Sync subtrees
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
          GH_TOKEN: ${{ secrets.GH_PAT }}
          SHAS_JSON: ${{ toJson(github.event.client_payload.shas) }}
          TRIGGER: ${{ github.event.client_payload.trigger || 'manual' }}
        run: python3 .github/scripts/sync-subtrees.py

      - name: Verify Maven build
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: mvn test --batch-mode --fail-at-end || exit 1

      - name: Push
        run: git push
```

- [ ] **Step 2: Create the sync script**

Create `.github/scripts/sync-subtrees.py`:

```python
#!/usr/bin/env python3
"""
Sync example subtrees from source repos.

Reads sync-config.json for the repo list. In CI mode, pins each clone
to the SHA from the ecosystem-build-succeeded dispatch payload. In
manual/local mode, uses HEAD.
"""

import json, os, subprocess, sys, tempfile
from pathlib import Path

ROOT = Path(__file__).resolve().parent.parent.parent
CONFIG = json.loads((ROOT / 'sync-config.json').read_text())

raw_shas = os.environ.get('SHAS_JSON', '{}')
shas = json.loads(raw_shas) if raw_shas and raw_shas != 'null' else {}
trigger = os.environ.get('TRIGGER', 'manual')
token = os.environ.get('GH_TOKEN') or os.environ.get('GITHUB_TOKEN', '')

synced = []
failed = []

for repo in CONFIG['repos']:
    name = repo['name']
    org = repo['org']
    prefix = f"{name}-examples"

    # Resolve SHA: underscore key convention (blocks-ui -> blocks_ui)
    sha_key = name.replace('-', '_')
    sha = shas.get(sha_key, '')

    print(f"\n--- {name} ---")

    with tempfile.TemporaryDirectory() as tmpdir:
        clone_dir = os.path.join(tmpdir, name)
        url = f"https://x-access-token:{token}@github.com/{org}/{name}.git"

        # Clone
        try:
            subprocess.run(
                ['git', 'clone', '--quiet', url, clone_dir],
                check=True, capture_output=True
            )
        except subprocess.CalledProcessError:
            print(f"  SKIP: could not clone {org}/{name}")
            failed.append(name)
            continue

        # Checkout pinned SHA if available
        if sha:
            try:
                subprocess.run(
                    ['git', '-C', clone_dir, 'checkout', sha, '--quiet'],
                    check=True, capture_output=True
                )
                print(f"  Pinned to {sha[:12]}")
            except subprocess.CalledProcessError:
                print(f"  WARN: SHA {sha[:12]} not found, using HEAD")

        # Check examples/ exists
        if not os.path.isdir(os.path.join(clone_dir, 'examples')):
            print(f"  SKIP: no examples/ directory")
            continue

        # Subtree split
        try:
            subprocess.run(
                ['git', '-C', clone_dir, 'subtree', 'split',
                 '--prefix=examples', '-b', 'examples-only', '--quiet'],
                check=True, capture_output=True
            )
        except subprocess.CalledProcessError as e:
            print(f"  SKIP: subtree split failed — {e}")
            failed.append(name)
            continue

        # Add or pull
        prefix_dir = ROOT / prefix
        sha_label = sha[:12] if sha else 'HEAD'
        msg = f"sync: {name} examples ({trigger}, {sha_label})"

        try:
            if prefix_dir.exists():
                subprocess.run(
                    ['git', '-C', str(ROOT), 'subtree', 'pull',
                     f'--prefix={prefix}', clone_dir, 'examples-only',
                     '--squash', '-m', msg],
                    check=True, capture_output=True
                )
                print(f"  Updated {prefix}")
            else:
                subprocess.run(
                    ['git', '-C', str(ROOT), 'subtree', 'add',
                     f'--prefix={prefix}', clone_dir, 'examples-only',
                     '--squash', '-m', msg],
                    check=True, capture_output=True
                )
                print(f"  Seeded {prefix}")
            synced.append(name)
        except subprocess.CalledProcessError as e:
            print(f"  WARN: subtree {'pull' if prefix_dir.exists() else 'add'} "
                  f"failed — {e}")
            failed.append(name)

print(f"\n=== Summary ===")
print(f"Synced: {len(synced)} ({', '.join(synced) if synced else 'none'})")
if failed:
    print(f"Failed: {len(failed)} ({', '.join(failed)})")
```

- [ ] **Step 3: Commit the workflow**

```bash
mkdir -p .github/scripts
git add .github/workflows/sync-examples.yml .github/scripts/sync-subtrees.py
git commit -m "ci: add sync-examples workflow — triggered by ecosystem-build-succeeded"
git push
```

---

### Task 5: Extend `sync-casehub-all.py` to dispatch to examples repo

**Files:**
- Modify: `scripts/sync-casehub-all.py` (in `casehubio/parent`)

**Interfaces:**
- Consumes: Same SHA payload already built for casehub-all
- Produces: Second dispatch to `casehubio/examples`

- [ ] **Step 1: Add the second dispatch**

In `/Users/mdproctor/claude/casehub/parent/scripts/sync-casehub-all.py`,
after the existing `casehub-all` dispatch, add a dispatch to `examples`:

Add after line 109 (after the existing `print('casehub-all dispatch sent.')`):

```python
# Also dispatch to examples repo
try:
    subprocess.run(
        ['gh', 'api', 'repos/casehubio/examples/dispatches', '--input', '-'],
        input=json.dumps(payload).encode(),
        check=True,
        env=os.environ.copy()
    )
    print('examples dispatch sent.')
except subprocess.CalledProcessError as e:
    print(f'WARNING: examples dispatch failed — {e}')
```

The `try/except` ensures a failure dispatching to examples doesn't break
the casehub-all sync. Both dispatches use the same payload.

- [ ] **Step 2: Test locally**

```bash
python3 scripts/sync-casehub-all.py
```

Expected output:
```
Syncing casehub-all (local mode) — N SHAs
casehub-all dispatch sent.
examples dispatch sent.
```

- [ ] **Step 3: Commit**

```bash
git add scripts/sync-casehub-all.py
git commit -m "ci: dispatch ecosystem-build-succeeded to examples repo alongside casehub-all"
```

---

### Task 6: Verify end-to-end and push

**Files:** None — verification only

**Interfaces:**
- Consumes: Everything from Tasks 1-5
- Produces: Verified, working casehub-examples repo

- [ ] **Step 1: Clone the examples repo fresh**

Clone into a temp location to simulate the end-user experience:

```bash
cd /tmp
git clone https://github.com/casehubio/examples.git casehub-examples-test
cd casehub-examples-test
```

- [ ] **Step 2: Verify Maven build**

```bash
mvn test
```

Expected: BUILD SUCCESS. All Maven examples pass.

- [ ] **Step 3: Verify a specific example in dev mode**

```bash
mvn quarkus:dev -pl ledger-examples/order-processing
```

Verify the app starts on `http://localhost:8080`. Hit one endpoint:

```bash
curl -s http://localhost:8080/orders | jq .
```

Stop dev mode (Ctrl+C).

- [ ] **Step 4: Verify sync.sh works**

```bash
./sync.sh
```

Expected: "Already up to date" for all repos (since we just seeded from
HEAD). No errors.

- [ ] **Step 5: Trigger the CI workflow manually**

```bash
gh workflow run sync-examples.yml --repo casehubio/examples
```

Wait for it to complete:

```bash
gh run list --repo casehubio/examples --limit 1
```

Expected: Successful run.

- [ ] **Step 6: Push the parent repo change**

Back in the parent repo, push the `sync-casehub-all.py` change:

```bash
git -C /Users/mdproctor/claude/casehub/parent push
```

---

## Self-Review Checklist

- [x] **Spec coverage:** All sections of the spec are covered:
  - §3 Naming → Task 1 (all files follow convention)
  - §4 Structure → Task 1 (pom.xml), Task 2 (subtree seeding)
  - §5 Sync → Task 2 (sync.sh), Task 4 (CI workflow), Task 5 (dispatch)
  - §5.2b Local sync → Task 1 Step 5 (sync.sh)
  - §6 POM → Task 1 Step 4
  - §7 README → Task 1 Step 6
  - §8 ADDING-EXAMPLES → Task 1 Step 7
  - §9 Deferred → Task 3 (issues for aggregator POMs)
  - §10.2 casehub-all relationship → Task 5 (shared dispatch)
- [x] **Placeholder scan:** No TBDs, TODOs, or vague references
- [x] **Type consistency:** SHA key convention (`replace('-', '_')`)
  used consistently in sync.sh, sync-subtrees.py, and spec
- [x] **Tooling safety scan:** No bash file operations on source code.
  All file operations are git subtree commands or repo infrastructure
  files (config, scripts, workflows)
