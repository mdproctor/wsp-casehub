# Documentation Audit, RAG Optimization & Freshness Gate — Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** TBD — to be created as epic
**Issue group:** TBD — per-repo audit issues created in Batch 4

**Goal:** Build detection tooling and enforcement gates that prevent documentation staleness, restructure guides for RAG-quality retrieval, and audit all 28 repos to bring docs current.

**Architecture:** Structural anchors in YAML frontmatter connect documentation sections to the code they describe. A Python detection script (shared between work-end and CI) flags candidate-stale sections when anchored elements change. A neocortex CLI module ingests anchored documentation into the RAG pipeline with per-section metadata for filtered retrieval.

**Tech Stack:** Python 3.11+ (detection script, bats-like tests), Java 21+ / Quarkus (neocortex module), GitHub Actions YAML, YAML frontmatter in Markdown

## Global Constraints

- Detection script must be pure Python (no external deps) — runs in any Claude session
- All code changes follow TDD: failing test → implement → verify
- Soredium is the source of truth for skills — edit there, sync-local to install
- Neocortex doc-ingestion module follows the example-rag-pipeline pattern exactly
- Guide decomposition follows the PLATFORM.md → platform/*.md precedent
- Structural anchors use the YAML frontmatter format defined in the spec §3.1
- Chunk ownership: child repos own their docs under `docs/guides/capabilities/` and `docs/guides/internals/`; parent aggregates via subtree

---

## Batch 1: Detection Infrastructure

**Repo:** `soredium` (skill source)
**After this batch:** A tested Python script can parse YAML frontmatter anchors from documentation files and detect which sections are candidate-stale given a git diff.

### Task 1: Structural Anchor Parser + Detection Script

**Files:**
- Create: `soredium/doc-freshness/doc_freshness_check.py`
- Create: `soredium/doc-freshness/test_doc_freshness_check.py`
- Create: `soredium/doc-freshness/SKILL.md` (minimal — just the script runner)

**Interfaces:**
- Produces: `parse_anchors(filepath) → dict` — parses YAML frontmatter, returns `{classes: [...], spis: [...], config_keys: [...], protocols: [...]}`
- Produces: `find_anchored_docs(docs_dir) → list[AnchoredDoc]` — scans docs dir for files with anchor frontmatter
- Produces: `detect_stale(diff_files, anchored_docs) → list[StaleCandidate]` — matches changed files against anchors
- Produces: `main(args) → JSON output` — CLI entry point for work-end and CI

- [ ] **Step 1: Write failing test for YAML frontmatter anchor parsing**

```python
# test_doc_freshness_check.py
import json
import os
import tempfile
import unittest

from doc_freshness_check import parse_anchors, AnchoredDoc


class TestParseAnchors(unittest.TestCase):

    def test_parses_full_anchor_frontmatter(self):
        content = """---
capability: notifications
audience: consumer
repo: casehub-platform
anchors:
  classes:
    - io.casehub.platform.notification.NotificationBridge
    - io.casehub.platform.notification.SubscriptionEngine
  spis:
    - io.casehub.platform.notification.spi.DeliveryChannel
  config-keys:
    - casehub.notification.digest.interval
  protocols:
    - notification-delivery-contract
---

# Notifications

Send notifications to users via multiple channels.
"""
        with tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False) as f:
            f.write(content)
            path = f.name
        try:
            result = parse_anchors(path)
            self.assertEqual(result['capability'], 'notifications')
            self.assertEqual(result['audience'], 'consumer')
            self.assertEqual(result['repo'], 'casehub-platform')
            self.assertIn('io.casehub.platform.notification.NotificationBridge',
                          result['anchors']['classes'])
            self.assertIn('io.casehub.platform.notification.spi.DeliveryChannel',
                          result['anchors']['spis'])
            self.assertIn('casehub.notification.digest.interval',
                          result['anchors']['config-keys'])
            self.assertIn('notification-delivery-contract',
                          result['anchors']['protocols'])
        finally:
            os.unlink(path)

    def test_no_frontmatter_returns_none(self):
        content = "# Just a heading\n\nNo frontmatter here.\n"
        with tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False) as f:
            f.write(content)
            path = f.name
        try:
            result = parse_anchors(path)
            self.assertIsNone(result)
        finally:
            os.unlink(path)

    def test_frontmatter_without_anchors_returns_metadata_only(self):
        content = """---
capability: notifications
audience: consumer
repo: casehub-platform
---

# Notifications
"""
        with tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False) as f:
            f.write(content)
            path = f.name
        try:
            result = parse_anchors(path)
            self.assertEqual(result['capability'], 'notifications')
            self.assertEqual(result['anchors'], {})
        finally:
            os.unlink(path)

    def test_verified_current_annotation_parsed(self):
        content = """---
capability: notifications
audience: consumer
repo: casehub-platform
verified-current: "2026-08-31 | commit:abc123"
anchors:
  classes:
    - io.casehub.platform.notification.NotificationBridge
---

# Notifications
"""
        with tempfile.NamedTemporaryFile(mode='w', suffix='.md', delete=False) as f:
            f.write(content)
            path = f.name
        try:
            result = parse_anchors(path)
            self.assertIn('2026-08-31', result['verified-current'])
        finally:
            os.unlink(path)
```

- [ ] **Step 2: Run test to verify it fails**

Run: `python3 -m pytest soredium/doc-freshness/test_doc_freshness_check.py -v`
Expected: FAIL with `ModuleNotFoundError: No module named 'doc_freshness_check'`

- [ ] **Step 3: Implement anchor parser**

```python
# doc_freshness_check.py
"""Documentation freshness detection — shared by work-end gate and CI Action."""

import json
import os
import re
import sys
from dataclasses import dataclass, field


def parse_anchors(filepath: str) -> dict | None:
    """Parse YAML frontmatter with structural anchors from a markdown file."""
    with open(filepath, 'r') as f:
        content = f.read()

    if not content.startswith('---'):
        return None

    close = content.find('\n---', 3)
    if close < 0:
        return None

    fm_block = content[4:close].strip()
    result = _parse_yaml_frontmatter(fm_block)
    result.setdefault('anchors', {})
    return result


def _parse_yaml_frontmatter(block: str) -> dict:
    """Minimal YAML parser for frontmatter — handles flat keys and one level of nesting."""
    result = {}
    current_key = None
    current_list_key = None

    for line in block.split('\n'):
        stripped = line.strip()
        if not stripped:
            continue

        indent = len(line) - len(line.lstrip())

        if indent == 0 and ':' in stripped:
            key, _, value = stripped.partition(':')
            key = key.strip()
            value = value.strip().strip('"').strip("'")
            if value:
                result[key] = value
                current_key = None
                current_list_key = None
            else:
                current_key = key
                if key not in result:
                    result[key] = {}
                current_list_key = None

        elif indent > 0 and current_key is not None:
            if stripped.startswith('- '):
                item = stripped[2:].strip()
                if current_list_key and current_key in result and isinstance(result[current_key], dict):
                    result[current_key].setdefault(current_list_key, [])
                    result[current_key][current_list_key].append(item)
            elif ':' in stripped:
                sub_key, _, sub_value = stripped.partition(':')
                sub_key = sub_key.strip()
                sub_value = sub_value.strip()
                if not sub_value:
                    current_list_key = sub_key
                    if isinstance(result.get(current_key), dict):
                        result[current_key].setdefault(sub_key, [])
                else:
                    if isinstance(result.get(current_key), dict):
                        result[current_key][sub_key] = sub_value

    return result


@dataclass
class AnchoredDoc:
    path: str
    capability: str
    audience: str
    repo: str
    anchors: dict = field(default_factory=dict)
    verified_current: str | None = None


def find_anchored_docs(docs_dir: str) -> list[AnchoredDoc]:
    """Scan a docs directory for files with structural anchor frontmatter."""
    results = []
    for root, _, files in os.walk(docs_dir):
        for fname in files:
            if not fname.endswith('.md'):
                continue
            fpath = os.path.join(root, fname)
            parsed = parse_anchors(fpath)
            if parsed and parsed.get('anchors'):
                results.append(AnchoredDoc(
                    path=os.path.relpath(fpath, docs_dir),
                    capability=parsed.get('capability', ''),
                    audience=parsed.get('audience', ''),
                    repo=parsed.get('repo', ''),
                    anchors=parsed.get('anchors', {}),
                    verified_current=parsed.get('verified-current'),
                ))
    return results


@dataclass
class StaleCandidate:
    doc_path: str
    capability: str
    anchor_type: str
    anchor_value: str
    changed_file: str
    reason: str


def detect_stale(diff_files: list[str], anchored_docs: list[AnchoredDoc]) -> list[StaleCandidate]:
    """Match changed files against structural anchors to find candidate-stale sections."""
    candidates = []

    for doc in anchored_docs:
        if doc.verified_current:
            continue

        for anchor_type, anchor_values in doc.anchors.items():
            for anchor in anchor_values:
                for changed in diff_files:
                    if _anchor_matches_file(anchor, anchor_type, changed):
                        candidates.append(StaleCandidate(
                            doc_path=doc.path,
                            capability=doc.capability,
                            anchor_type=anchor_type,
                            anchor_value=anchor,
                            changed_file=changed,
                            reason=f"{anchor_type} anchor '{anchor}' matches changed file '{changed}'"
                        ))

    seen = set()
    deduped = []
    for c in candidates:
        key = (c.doc_path, c.anchor_value)
        if key not in seen:
            seen.add(key)
            deduped.append(c)

    return deduped


def _anchor_matches_file(anchor: str, anchor_type: str, changed_file: str) -> bool:
    """Check if a structural anchor matches a changed file path."""
    if anchor_type == 'classes' or anchor_type == 'spis':
        simple_name = anchor.rsplit('.', 1)[-1]
        expected_path = anchor.replace('.', '/') + '.java'
        alt_path = anchor.replace('.', '/') + '.kt'
        return changed_file.endswith(expected_path) or changed_file.endswith(alt_path)

    elif anchor_type == 'config-keys':
        config_files = ('application.properties', 'application.yaml', 'application.yml')
        return any(changed_file.endswith(cf) for cf in config_files)

    elif anchor_type == 'protocols':
        return anchor in changed_file

    return False


def main():
    """CLI entry point. Usage: doc-freshness-check.py --diff <file> --docs <dir> [--graph <file>]"""
    import argparse
    parser = argparse.ArgumentParser(description='Documentation freshness detection')
    parser.add_argument('--diff', required=True, help='File containing list of changed files (one per line)')
    parser.add_argument('--docs', required=True, help='Path to docs directory')
    parser.add_argument('--graph', help='Path to dependency-graph.json (for dependent repo checks)')
    parser.add_argument('--repo', help='Current repo name (for cross-repo filtering)')
    args = parser.parse_args()

    with open(args.diff) as f:
        diff_files = [line.strip() for line in f if line.strip()]

    anchored = find_anchored_docs(args.docs)
    candidates = detect_stale(diff_files, anchored)

    output = {
        'candidates': [
            {
                'doc_path': c.doc_path,
                'capability': c.capability,
                'anchor_type': c.anchor_type,
                'anchor_value': c.anchor_value,
                'changed_file': c.changed_file,
                'reason': c.reason,
            }
            for c in candidates
        ],
        'total_anchored_docs': len(anchored),
        'total_candidates': len(candidates),
    }

    print(json.dumps(output, indent=2))
    return 1 if candidates else 0


if __name__ == '__main__':
    sys.exit(main())
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `python3 -m pytest soredium/doc-freshness/test_doc_freshness_check.py -v`
Expected: All 4 tests PASS

- [ ] **Step 5: Write detection tests**

```python
# Add to test_doc_freshness_check.py

class TestDetectStale(unittest.TestCase):

    def _make_doc(self, path='notifications.md', classes=None, spis=None,
                  config_keys=None, protocols=None, verified=None):
        return AnchoredDoc(
            path=path, capability='notifications', audience='consumer',
            repo='casehub-platform',
            anchors={
                k: v for k, v in [
                    ('classes', classes or []),
                    ('spis', spis or []),
                    ('config-keys', config_keys or []),
                    ('protocols', protocols or []),
                ] if v
            },
            verified_current=verified,
        )

    def test_class_change_flags_section(self):
        doc = self._make_doc(classes=[
            'io.casehub.platform.notification.NotificationBridge'
        ])
        diff = ['platform/src/main/java/io/casehub/platform/notification/NotificationBridge.java']
        result = detect_stale(diff, [doc])
        self.assertEqual(len(result), 1)
        self.assertEqual(result[0].anchor_type, 'classes')

    def test_unrelated_change_no_flag(self):
        doc = self._make_doc(classes=[
            'io.casehub.platform.notification.NotificationBridge'
        ])
        diff = ['engine/src/main/java/io/casehub/engine/CaseEngine.java']
        result = detect_stale(diff, [doc])
        self.assertEqual(len(result), 0)

    def test_verified_current_suppresses_flag(self):
        doc = self._make_doc(
            classes=['io.casehub.platform.notification.NotificationBridge'],
            verified='2026-08-31 | commit:abc123'
        )
        diff = ['platform/src/main/java/io/casehub/platform/notification/NotificationBridge.java']
        result = detect_stale(diff, [doc])
        self.assertEqual(len(result), 0)

    def test_spi_change_flags_section(self):
        doc = self._make_doc(spis=[
            'io.casehub.platform.notification.spi.DeliveryChannel'
        ])
        diff = ['platform-api/src/main/java/io/casehub/platform/notification/spi/DeliveryChannel.java']
        result = detect_stale(diff, [doc])
        self.assertEqual(len(result), 1)
        self.assertEqual(result[0].anchor_type, 'spis')

    def test_config_key_change_flags_section(self):
        doc = self._make_doc(config_keys=['casehub.notification.digest.interval'])
        diff = ['src/main/resources/application.properties']
        result = detect_stale(diff, [doc])
        self.assertEqual(len(result), 1)

    def test_deduplicates_multiple_file_matches(self):
        doc = self._make_doc(classes=[
            'io.casehub.platform.notification.NotificationBridge'
        ])
        diff = [
            'platform/src/main/java/io/casehub/platform/notification/NotificationBridge.java',
            'platform/src/test/java/io/casehub/platform/notification/NotificationBridgeTest.java',
        ]
        result = detect_stale(diff, [doc])
        self.assertEqual(len(result), 1)


class TestAnchorIntegrity(unittest.TestCase):

    def test_find_anchored_docs_in_directory(self):
        with tempfile.TemporaryDirectory() as tmpdir:
            cap_dir = os.path.join(tmpdir, 'capabilities')
            os.makedirs(cap_dir)
            with open(os.path.join(cap_dir, 'notifications.md'), 'w') as f:
                f.write("""---
capability: notifications
audience: consumer
repo: casehub-platform
anchors:
  classes:
    - io.casehub.platform.notification.NotificationBridge
---

# Notifications
""")
            with open(os.path.join(cap_dir, 'plain.md'), 'w') as f:
                f.write("# No frontmatter\n")

            docs = find_anchored_docs(tmpdir)
            self.assertEqual(len(docs), 1)
            self.assertEqual(docs[0].capability, 'notifications')


if __name__ == '__main__':
    unittest.main()
```

- [ ] **Step 6: Run all tests**

Run: `python3 -m pytest soredium/doc-freshness/test_doc_freshness_check.py -v`
Expected: All 11 tests PASS

- [ ] **Step 7: Write minimal SKILL.md**

```markdown
---
name: doc-freshness
description: >
  Documentation freshness detection — structural anchor check for work-end
  gate and CI enforcement. Not invoked directly; called by work-end
  orchestrator's doc_freshness_gate step.
slash-command: false
---

# Doc Freshness Check

Internal skill — not user-invoked. Called by work-end orchestrator.

## Usage

The detection script is called with:

\`\`\`bash
python3 ~/.claude/skills/doc-freshness/doc_freshness_check.py \
    --diff <changed-files-list> --docs <docs-dir> [--graph <dep-graph>]
\`\`\`

Output: JSON with candidate-stale sections. Exit code 1 if candidates found, 0 if clean.
```

- [ ] **Step 8: Commit**

```bash
git -C ~/claude/hortora/soredium add doc-freshness/
git -C ~/claude/hortora/soredium commit -m "feat: doc-freshness detection script with structural anchor parsing

Parses YAML frontmatter anchors (classes, SPIs, config keys, protocols)
from documentation files. Matches git diff against anchors to identify
candidate-stale documentation sections. Shared by work-end gate and CI.

Refs casehubio/parent#TBD

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 2: Gate & CI Enforcement

**Repos:** `soredium` (work-end), `parent` (GitHub Actions)
**After this batch:** Work-end blocks on stale docs (advisory mode initially). CI checks anchor integrity on every PR.

### Task 2: Work-End Orchestrator Gate Step

**Files:**
- Modify: `soredium/work-end/work_end_orchestrator.py` — add `doc_freshness_gate` StepDef
- Create: `soredium/work-end/handlers/doc_freshness.md` — handler instructions
- Modify: `soredium/work-end/SKILL.md` — add action dispatch entry

**Interfaces:**
- Consumes: `doc_freshness_check.py` from Task 1 (installed at `~/.claude/skills/doc-freshness/`)
- Produces: `doc_freshness_gate` step in work-end orchestrator pipeline

- [ ] **Step 1: Add StepDef to orchestrator**

In `work_end_orchestrator.py`, add after the `impl_doc_sync` StepDef (line ~811):

```python
    StepDef("doc_freshness_gate", "closing:review", "judgment",
            skip_fn=_is_sweep_deselected("doc_freshness_gate")),
```

Also add `"doc_freshness_gate"` to the `SWEEP_STEPS` list (line 68) and `PER_REPO_SWEEP_STEPS` set (line 135).

- [ ] **Step 2: Add action dispatch entry in SKILL.md**

Add to the ACTION dispatch table in work-end SKILL.md:

```markdown
| `doc_freshness_gate` | Read `handlers/doc_freshness.md` |
```

- [ ] **Step 3: Write handler**

```markdown
# handlers/doc_freshness.md

## doc_freshness_gate

Documentation freshness check — detect candidate-stale sections via
structural anchors.

### Step 1 — Generate diff file list

\`\`\`bash
git -C $PROJECT diff --name-only $BASE_BRANCH...HEAD > /tmp/doc-freshness-diff.txt
\`\`\`

### Step 2 — Run detection

\`\`\`bash
python3 ~/.claude/skills/doc-freshness/doc_freshness_check.py \
    --diff /tmp/doc-freshness-diff.txt \
    --docs $PROJECT/docs
\`\`\`

### Step 3 — Interpret results

If `total_candidates` is 0: report `step_done=doc_freshness_gate produced=0`.

If candidates found:
1. Print each candidate with its doc path, anchor, and changed file
2. For each candidate, read the documentation section and the changed code
3. Determine if the section needs updating (adversarial check)
4. If updates needed: update the section inline, commit, then report done
5. If section is current despite anchor change: add `verified-current` annotation

**Advisory mode (pre-activation):** Report findings but do not block.
Print: "Doc freshness gate (advisory): N candidate-stale sections found"
then report `step_done=doc_freshness_gate produced=N`.
```

- [ ] **Step 4: Commit**

```bash
git -C ~/claude/hortora/soredium add work-end/
git -C ~/claude/hortora/soredium commit -m "feat: doc_freshness_gate step in work-end orchestrator

Advisory mode — reports candidate-stale sections but does not block.
Hard gate activates after validation corpus confirms ≥80% precision.

Refs casehubio/parent#TBD

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 3: GitHub Actions — Dependency Graph & Doc Freshness

**Files:**
- Create: `parent/.github/workflows/dependency-graph.yml` — daily POM analysis
- Create: `parent/.github/workflows/doc-freshness.yml` — reusable PR workflow
- Create: `parent/scripts/generate-dependency-graph.py` — POM → JSON script

**Interfaces:**
- Produces: `dependency-graph.json` artifact in parent repo (consumed by work-end gate)
- Produces: Reusable workflow callable from child repos

- [ ] **Step 1: Write dependency graph generation script**

```python
# scripts/generate-dependency-graph.py
"""Generate dependency-graph.json from Maven POM analysis across all CaseHub repos."""

import json
import os
import subprocess
import sys
import xml.etree.ElementTree as ET

REPOS = [
    'platform', 'worker', 'ledger', 'connectors', 'work', 'qhorus',
    'eidos', 'neocortex', 'engine', 'iot', 'ras', 'desiredstate',
    'blocks', 'blocks-ui', 'claudony', 'openclaw', 'workers', 'ops',
    'pages', 'devtown', 'aml', 'clinical', 'life', 'drafthouse',
    'quarkmind', 'soc', 'fsitrading', 'chat-app',
]

NS = {'m': 'http://maven.apache.org/POM/4.0.0'}


def extract_dependencies(pom_path: str) -> list[str]:
    """Extract casehub dependency artifactIds from a POM file."""
    try:
        tree = ET.parse(pom_path)
    except ET.ParseError:
        return []
    root = tree.getroot()
    deps = []
    for dep in root.findall('.//m:dependency', NS):
        group = dep.findtext('m:groupId', '', NS)
        artifact = dep.findtext('m:artifactId', '', NS)
        if group.startswith('io.casehub'):
            deps.append(artifact)
    return deps


def build_graph(repos_root: str) -> dict:
    """Build dependency graph from all repo POMs."""
    graph = {}
    for repo in REPOS:
        repo_dir = os.path.join(repos_root, repo)
        if not os.path.isdir(repo_dir):
            continue
        all_deps = set()
        for root, _, files in os.walk(repo_dir):
            for f in files:
                if f == 'pom.xml':
                    deps = extract_dependencies(os.path.join(root, f))
                    all_deps.update(deps)
        graph[f'casehub-{repo}'] = {
            'depends_on': sorted(all_deps),
            'depended_on_by': [],
        }

    for repo, data in graph.items():
        for dep in data['depends_on']:
            if dep in graph:
                graph[dep]['depended_on_by'].append(repo)

    return graph


if __name__ == '__main__':
    repos_root = sys.argv[1] if len(sys.argv) > 1 else os.path.expanduser('~/claude/casehub')
    graph = build_graph(repos_root)
    output_path = os.path.join(os.path.dirname(__file__), '..', 'dependency-graph.json')
    with open(output_path, 'w') as f:
        json.dump(graph, f, indent=2)
    print(f"Generated dependency graph: {len(graph)} repos, "
          f"{sum(len(d['depends_on']) for d in graph.values())} edges")
```

- [ ] **Step 2: Write dependency graph Action**

```yaml
# .github/workflows/dependency-graph.yml
name: Generate Dependency Graph

on:
  schedule:
    - cron: '0 6 * * *'
  workflow_dispatch:

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Clone child repos
        run: |
          mkdir -p /tmp/casehub
          for repo in platform worker ledger connectors work qhorus eidos neocortex engine iot ras desiredstate blocks blocks-ui claudony openclaw workers ops pages devtown aml clinical life drafthouse quarkmind soc fsitrading chat-app; do
            gh repo clone casehubio/$repo /tmp/casehub/$repo -- --depth 1 2>/dev/null || true
          done
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}

      - name: Generate graph
        run: python3 scripts/generate-dependency-graph.py /tmp/casehub

      - name: Commit if changed
        run: |
          git add dependency-graph.json
          git diff --cached --quiet || git commit -m "chore: update dependency-graph.json [skip ci]"
          git push
        env:
          GH_TOKEN: ${{ secrets.GH_PAT }}
```

- [ ] **Step 3: Write doc freshness reusable workflow**

```yaml
# .github/workflows/doc-freshness.yml
name: Doc Freshness Check

on:
  workflow_call:

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0

      - name: Checkout parent for detection script
        uses: actions/checkout@v4
        with:
          repository: casehubio/parent
          path: .parent
          token: ${{ secrets.GH_PAT }}

      - name: Generate diff file list
        run: |
          git diff --name-only origin/${{ github.base_ref }}...HEAD > /tmp/diff-files.txt

      - name: Run doc freshness check
        run: |
          python3 .parent/scripts/doc-freshness-check-ci.py \
            --diff /tmp/diff-files.txt \
            --docs docs/ \
            --repo ${{ github.event.repository.name }}

      - name: Check anchor integrity
        run: |
          python3 .parent/scripts/check-anchor-integrity.py \
            --docs docs/ \
            --repo ${{ github.event.repository.name }} \
            --source-root .
```

- [ ] **Step 4: Write CI-specific wrapper scripts**

Create `scripts/doc-freshness-check-ci.py` — thin wrapper that calls the detection logic and posts PR comments.

Create `scripts/check-anchor-integrity.py` — walks docs, extracts anchors, verifies each class/SPI resolves in the codebase via `git ls-files`.

These scripts import the core logic from `doc_freshness_check.py` (vendored into parent's scripts/ for CI access).

- [ ] **Step 5: Commit**

```bash
git -C ~/claude/casehub/parent add .github/workflows/ scripts/ dependency-graph.json
git -C ~/claude/casehub/parent commit -m "feat: dependency graph Action + doc freshness reusable workflow

Daily CI generates dependency-graph.json from POM analysis.
Reusable doc freshness workflow checks anchor integrity on PRs.

Refs #TBD

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 3: RAG Pipeline

**Repo:** `neocortex`
**After this batch:** Documentation with YAML frontmatter can be ingested into the RAG pipeline with per-section metadata for filtered retrieval.

### Task 4: DocMetadataExtractor + Doc Ingestion CLI Module

**Files:**
- Create: `neocortex/doc-ingestion/pom.xml`
- Create: `neocortex/doc-ingestion/src/main/java/io/casehub/neocortex/doc/DocMetadataExtractor.java`
- Create: `neocortex/doc-ingestion/src/main/java/io/casehub/neocortex/doc/DocIngestionCli.java`
- Create: `neocortex/doc-ingestion/src/test/java/io/casehub/neocortex/doc/DocMetadataExtractorTest.java`
- Modify: `neocortex/pom.xml` — add `doc-ingestion` module

**Interfaces:**
- Consumes: `MetadataExtractor` SPI from `rag-api`
- Consumes: `CorpusIngestionService`, `CorpusIngestionBinding` from `rag`
- Consumes: `FlatChangeSource`, `FlatCorpusStore` from `corpus`
- Produces: `DocMetadataExtractor` — `MetadataExtractor` impl that parses nested YAML anchors into `listMetadata`
- Produces: `DocIngestionCli` — `@QuarkusMain` CLI tool for batch ingestion

- [ ] **Step 1: Write failing test for DocMetadataExtractor**

```java
package io.casehub.neocortex.doc;

import io.casehub.neocortex.rag.ExtractionResult;
import org.junit.jupiter.api.Test;

import java.nio.charset.StandardCharsets;

import static org.junit.jupiter.api.Assertions.*;

class DocMetadataExtractorTest {

    private final DocMetadataExtractor extractor = new DocMetadataExtractor();

    @Test
    void parsesCapabilityAndAudienceFromFrontmatter() {
        String content = """
                ---
                capability: notifications
                audience: consumer
                repo: casehub-platform
                anchors:
                  classes:
                    - io.casehub.platform.notification.NotificationBridge
                    - io.casehub.platform.notification.SubscriptionEngine
                  spis:
                    - io.casehub.platform.notification.spi.DeliveryChannel
                ---
                
                # Notifications
                
                Content here.
                """;

        ExtractionResult result = extractor.extract("notifications.md",
                content.getBytes(StandardCharsets.UTF_8));

        assertEquals("notifications", result.metadata().get("capability"));
        assertEquals("consumer", result.metadata().get("audience"));
        assertEquals("casehub-platform", result.metadata().get("repo"));
        assertTrue(result.listMetadata().containsKey("anchors.classes"));
        assertTrue(result.listMetadata().get("anchors.classes")
                .contains("io.casehub.platform.notification.NotificationBridge"));
        assertTrue(result.listMetadata().get("anchors.spis")
                .contains("io.casehub.platform.notification.spi.DeliveryChannel"));
        assertTrue(result.body().contains("# Notifications"));
    }

    @Test
    void noFrontmatterReturnsSameBody() {
        String content = "# Just a heading\n\nNo frontmatter.\n";
        ExtractionResult result = extractor.extract("plain.md",
                content.getBytes(StandardCharsets.UTF_8));
        assertEquals(content, result.body());
        assertTrue(result.metadata().isEmpty());
    }

    @Test
    void flatFrontmatterWithoutAnchorsStillParsesMetadata() {
        String content = """
                ---
                capability: notifications
                audience: consumer
                ---
                
                # Notifications
                """;
        ExtractionResult result = extractor.extract("notifications.md",
                content.getBytes(StandardCharsets.UTF_8));
        assertEquals("notifications", result.metadata().get("capability"));
        assertTrue(result.listMetadata().isEmpty());
    }
}
```

- [ ] **Step 2: Run test to verify it fails**

Run: `mvn -C neocortex test -pl doc-ingestion -Dtest=DocMetadataExtractorTest -am`
Expected: FAIL — class not found

- [ ] **Step 3: Implement DocMetadataExtractor**

```java
package io.casehub.neocortex.doc;

import io.casehub.neocortex.rag.ExtractionResult;
import io.casehub.neocortex.rag.MetadataExtractor;

import java.nio.charset.StandardCharsets;
import java.util.*;

public class DocMetadataExtractor implements MetadataExtractor {

    @Override
    public ExtractionResult extract(String path, byte[] content) {
        String text = new String(content, StandardCharsets.UTF_8);
        if (!text.startsWith("---")) {
            return new ExtractionResult(text, Map.of());
        }

        int close = text.indexOf("\n---", 3);
        if (close < 0) {
            return new ExtractionResult(text, Map.of());
        }

        String fmBlock = text.substring(4, close).trim();
        String body = text.substring(close + 4).trim();

        Map<String, String> metadata = new LinkedHashMap<>();
        Map<String, List<String>> listMetadata = new LinkedHashMap<>();
        parseYaml(fmBlock, metadata, listMetadata);

        return new ExtractionResult(body, metadata, listMetadata);
    }

    private void parseYaml(String block, Map<String, String> metadata,
                           Map<String, List<String>> listMetadata) {
        String topKey = null;
        String subKey = null;

        for (String line : block.split("\n")) {
            String stripped = line.strip();
            if (stripped.isEmpty()) continue;

            int indent = line.length() - line.stripLeading().length();

            if (indent == 0 && stripped.contains(":")) {
                int colon = stripped.indexOf(':');
                String key = stripped.substring(0, colon).strip();
                String value = stripped.substring(colon + 1).strip();
                value = stripQuotes(value);

                if (value.isEmpty()) {
                    topKey = key;
                    subKey = null;
                } else {
                    metadata.put(key, value);
                    topKey = null;
                    subKey = null;
                }
            } else if (indent > 0 && topKey != null) {
                if (stripped.startsWith("- ")) {
                    String item = stripped.substring(2).strip();
                    if (subKey != null) {
                        String listKey = topKey + "." + subKey;
                        listMetadata.computeIfAbsent(listKey, k -> new ArrayList<>()).add(item);
                    }
                } else if (stripped.contains(":")) {
                    int colon = stripped.indexOf(':');
                    String key = stripped.substring(0, colon).strip();
                    String value = stripped.substring(colon + 1).strip();
                    if (value.isEmpty()) {
                        subKey = key;
                    } else {
                        metadata.put(topKey + "." + key, stripQuotes(value));
                    }
                }
            }
        }
    }

    private String stripQuotes(String value) {
        if (value.length() >= 2 &&
                ((value.startsWith("\"") && value.endsWith("\"")) ||
                 (value.startsWith("'") && value.endsWith("'")))) {
            return value.substring(1, value.length() - 1);
        }
        return value;
    }
}
```

- [ ] **Step 4: Run tests to verify they pass**

Run: `mvn -C neocortex test -pl doc-ingestion -Dtest=DocMetadataExtractorTest -am`
Expected: All 3 tests PASS

- [ ] **Step 5: Write DocIngestionCli**

```java
package io.casehub.neocortex.doc;

import dev.langchain4j.data.document.DocumentSplitter;
import dev.langchain4j.data.document.splitter.DocumentSplitters;
import io.casehub.neocortex.corpus.zip.FlatChangeSource;
import io.casehub.neocortex.corpus.zip.FlatCorpusStore;
import io.casehub.neocortex.rag.CorpusRef;
import io.casehub.neocortex.rag.CursorStore;
import io.casehub.neocortex.rag.EmbeddingIngestor;
import io.casehub.neocortex.rag.runtime.CorpusIngestionBinding;
import io.casehub.neocortex.rag.runtime.CorpusIngestionService;
import io.quarkus.runtime.QuarkusApplication;
import io.quarkus.runtime.annotations.QuarkusMain;
import jakarta.inject.Inject;
import picocli.CommandLine;

import java.nio.file.Path;

@QuarkusMain
@CommandLine.Command(name = "doc-ingestion", mixinStandardHelpOptions = true)
public class DocIngestionCli implements QuarkusApplication, Runnable {

    @CommandLine.Option(names = "--docs", required = true,
            description = "Path to documentation directory")
    Path docsDir;

    @CommandLine.Option(names = "--tenant", defaultValue = "casehub",
            description = "Tenant ID for corpus isolation")
    String tenantId;

    @CommandLine.Option(names = "--reconcile", defaultValue = "false",
            description = "Run full reconciliation instead of incremental")
    boolean reconcile;

    @Inject EmbeddingIngestor ingestor;
    @Inject CursorStore cursorStore;

    @Override
    public int run(String... args) {
        return new CommandLine(this).execute(args);
    }

    @Override
    public void run() {
        var store = new FlatCorpusStore(docsDir);
        var changeSource = new FlatChangeSource(store, docsDir);
        var binding = new CorpusIngestionBinding(
                "casehub-docs",
                new CorpusRef(tenantId, "casehub-docs"),
                changeSource,
                store,
                new DocMetadataExtractor()
        );

        var service = new CorpusIngestionService(ingestor, cursorStore);
        DocumentSplitter splitter = DocumentSplitters.recursive(500, 50);

        if (reconcile) {
            service.reconcile("casehub-docs", binding, splitter);
            System.out.println("Reconciliation complete.");
        } else {
            service.processBinding(binding, splitter);
            System.out.println("Ingestion complete.");
        }
    }
}
```

- [ ] **Step 6: Create POM**

Create `doc-ingestion/pom.xml` with dependencies on `rag`, `corpus`, `rag-api`, Quarkus runtime, and picocli. Follow the `example-rag-pipeline/pom.xml` structure.

- [ ] **Step 7: Add module to parent POM**

Add `<module>doc-ingestion</module>` to `neocortex/pom.xml`.

- [ ] **Step 8: Run build**

Run: `mvn -C neocortex install -pl doc-ingestion -am -DskipTests=false`
Expected: BUILD SUCCESS, all tests pass

- [ ] **Step 9: Commit**

```bash
git -C ~/claude/casehub/neocortex add doc-ingestion/ pom.xml
git -C ~/claude/casehub/neocortex commit -m "feat: doc-ingestion CLI module for RAG pipeline

DocMetadataExtractor parses nested YAML frontmatter anchors into
ExtractionResult.listMetadata for filtered retrieval via PayloadFilter.
QuarkusMain CLI tool for batch ingestion following FlatCorpusIngestDemo.

Refs casehubio/parent#TBD

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## Batch 4: Doc Restructuring & Audit Launch

**Repo:** `parent` (docs), child repos (guide decomposition)
**After this batch:** Capability index exists, platform consumer guide is decomposed as template, audit issues created for all 28 repos.

### Task 5: Capability Index + Template Guide Decomposition

**Files:**
- Create: `parent/docs/capabilities.md` — cross-repo capability index
- Create: `parent/docs/repos/casehub-platform/capabilities/notifications.md` — template chunk
- Create: `parent/docs/repos/casehub-platform/capabilities/identity.md` — template chunk
- Create: `parent/docs/repos/casehub-platform/capabilities/expressions.md` — template chunk
- Modify: `parent/docs/repos/casehub-platform/consumer-guide.md` — thin index linking to chunks
- Modify: `parent/docs/INDEX.md` — add link to capabilities.md

**Interfaces:**
- Consumes: Existing consumer-guide.md content (418 lines for platform)
- Produces: Template for how all guides should be decomposed

- [ ] **Step 1: Read platform consumer guide and identify sections >40 lines**

Read `docs/repos/casehub-platform/consumer-guide.md`. Identify the largest sections by line count. Extract sections exceeding 40 lines into standalone capability chunks.

- [ ] **Step 2: Create capability index skeleton**

Create `docs/capabilities.md` with the cross-repo capability table. Populate with all known capabilities from the existing consumer-index.md, organized by domain. Link consumer column to `repos/<repo>/capabilities/<topic>.md` paths (even if chunks don't exist yet — they'll be created during the audit).

- [ ] **Step 3: Extract platform notification section to chunk**

Create `docs/repos/casehub-platform/capabilities/notifications.md` with:
- YAML frontmatter (capability, audience, repo, anchors)
- Self-contained content extracted from consumer-guide § Notifications
- All structural anchors referencing actual current classes/SPIs

- [ ] **Step 4: Extract 2-3 more platform sections as template chunks**

Repeat for identity and expressions sections. Each chunk gets its own YAML frontmatter with structural anchors.

- [ ] **Step 5: Convert consumer-guide.md to thin index**

Replace extracted sections with links to capability chunks. Keep the brief repo overview and key types summary. The guide becomes ~50-80 lines instead of 418.

- [ ] **Step 6: Update INDEX.md**

Add link to `docs/capabilities.md` in the cross-cutting section.

- [ ] **Step 7: Commit**

```bash
git -C ~/claude/casehub/parent add docs/
git -C ~/claude/casehub/parent commit -m "feat: capability index + platform guide decomposition template

Cross-repo capabilities.md routes by capability to individual chunks.
Platform consumer guide decomposed: notifications, identity, expressions
extracted as standalone chunks with structural anchor frontmatter.

Refs #TBD

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

### Task 6: Audit Issue Creation + Process Template

**Files:**
- Create: `parent/docs/audit/audit-process.md` — per-repo audit process template
- No code — GitHub issue creation via `gh`

**Interfaces:**
- Consumes: Staleness inventory (commit counts from context gathering)
- Produces: 28 GitHub issues with specific audit scope per repo

- [ ] **Step 1: Write audit process template**

Create `docs/audit/audit-process.md` documenting the per-repo audit process:
1. Delta analysis
2. Source mining (blogs, issues, commits)
3. Guide update
4. Structural anchor insertion
5. Guide decomposition (sections >40 lines)
6. Arc42stories 3-check sweep
7. Exit criteria verification

- [ ] **Step 2: Create epic issue**

```bash
gh issue create --repo casehubio/parent \
    --title "Documentation audit — RAG optimization & freshness gate" \
    --body "$(cat <<'EOF'
## Summary
Comprehensive audit of all 28 repo documentation: consumer guides, contributor
guides, arc42stories. 1,700+ commits of drift since last sync (Aug 3-10).

## Deliverables
- [ ] All guides updated to reflect current code
- [ ] Structural anchors added to all guide sections
- [ ] Guides >40 lines decomposed into capability chunks
- [ ] capabilities.md populated with all capabilities
- [ ] Work-end freshness gate active (advisory → hard after validation)
- [ ] CI doc freshness Action deployed to all repos

## Design spec
specs/doc-audit/2026-08-31-doc-audit-rag-optimization-design.md
EOF
)" --label "documentation,epic"
```

- [ ] **Step 3: Create per-repo audit issues**

Create one issue per repo on `casehubio/parent` (audit coordination repo), with specific scope:

For each repo, include: commit count since last sync, list of changed modules/SPIs from git log, arc42stories existence, blog entry count for source material.

Wave 1 (foundation, highest priority): platform, worker, ledger, connectors, work, qhorus, eidos, neocortex, engine, iot
Wave 2 (orchestration): ras, desiredstate, blocks, blocks-ui, claudony, openclaw, workers, ops, pages
Wave 3 (application): devtown, aml, clinical, life, drafthouse, quarkmind, soc, fsitrading, chat-app

- [ ] **Step 4: Commit process template**

```bash
git -C ~/claude/casehub/parent add docs/audit/
git -C ~/claude/casehub/parent commit -m "docs: audit process template + per-repo issues created

28 per-repo audit issues created, prioritized by dependency blast radius.
Process template documents the 7-step per-repo audit workflow.

Refs #TBD

Co-Authored-By: Claude Opus 4.6 (1M context) <noreply@anthropic.com>"
```

---

## References

- [2026-08-31-doc-audit-rag-optimization-design.md] — design spec this plan implements
- [decisions.md] — 7 validated decisions (D1-D7)
- [neocortex/rag/src/main/java/.../CorpusIngestionService.java:35] — ingestion service API
- [neocortex/rag/src/main/java/.../YamlFrontmatterExtractor.java:14] — base extractor pattern
- [neocortex/examples/example-rag-pipeline/src/main/java/.../FlatCorpusIngestDemo.java:16] — CLI precedent
- [neocortex/rag-api/src/main/java/.../ExtractionResult.java:6] — listMetadata record
- [soredium/work-end/work_end_orchestrator.py:811] — impl_doc_sync insertion point
- [GE-20260601-85afd0] — 3-check quality sweep for arc42stories
- [docs/platform/] — PLATFORM.md decomposition precedent
