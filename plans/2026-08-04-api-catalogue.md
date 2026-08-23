# API Catalogue Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> subagent-driven-development (recommended) or executing-plans to
> implement this plan task-by-task. Each task follows TDD
> (test-driven-development) and uses ide-tooling for structural
> editing. Steps use checkbox (`- [ ]`) syntax for tracking.

**Focal issue:** #402 — Auto-generated SPI-chunked API catalogue
**Issue group:** #402

**Goal:** Deliver an end-to-end pipeline that generates markdown API references from Java/TypeScript source, aggregates them in parent, and produces a cross-repo SPI implementation matrix.

**Architecture:** Existing doclet tools (jmarkdoc or neuhalje for Java, TypeDoc for TS) generate per-repo API docs into `docs/guides/api/`. Parent aggregates via existing subtree sync. A custom Python script in parent correlates SPI implementations across repos. This plan delivers the pipeline with engine (Java) and pages (TS) as pilots; rollout to remaining repos is mechanical and deferred.

**Tech Stack:** Java doclet (jmarkdoc or neuhalje), TypeDoc + typedoc-plugin-markdown, Python 3 (cross-repo overlay script), GitHub Actions, bash (sync-guides.sh)

## Global Constraints

- Generated output must be deterministic — identical input → identical output, byte-for-byte
- Generated files carry `<!-- Generated — do not edit -->` headers
- All automated commits use `[skip ci]` in the commit message
- CI triggers are path-filtered to source directories only (prevent loops)
- `docs/guides/api/` is deleted and recreated on each generation run (clean slate)

---

### Task 1: Evaluate Java Doclets

Evaluate jmarkdoc and neuhalje/markdown-doclet against engine-api. Pick one based on output quality, determinism, type hierarchy inclusion, and JDK compatibility.

**Files:**
- Read: `/Users/mdproctor/claude/casehub/engine/api/pom.xml`
- Read: engine-api source (via IntelliJ)
- Create: `/tmp/doclet-eval/` (temporary evaluation output)

**Interfaces:**
- Produces: chosen doclet tool name, configuration approach (Maven plugin vs standalone JAR)

- [ ] **Step 1: Check JDK version available**

```bash
java -version
```

If JDK 25+ is available, both tools are candidates. If only JDK 21, jmarkdoc is ruled out (requires JDK 25+ to run).

- [ ] **Step 2: Clone and build jmarkdoc (if JDK 25+ available)**

```bash
git clone https://github.com/AdamBien/jmarkdoc.git /tmp/jmarkdoc
```

Follow the build instructions in its README. If build fails or requires tooling not available, note and move on.

- [ ] **Step 3: Run jmarkdoc against engine-api**

```bash
# Run jmarkdoc pointing at engine-api source
# Output to /tmp/doclet-eval/jmarkdoc/
```

Inspect the output:
- Are method signatures complete with generics?
- Is javadoc included?
- Does it include implements/extends information?
- Run twice — is output identical? (`diff -r` between runs)

- [ ] **Step 4: Run neuhalje/markdown-doclet against engine-api**

Add to engine-api's pom.xml (temporary — don't commit):

```xml
<plugin>
  <groupId>org.apache.maven.plugins</groupId>
  <artifactId>maven-javadoc-plugin</artifactId>
  <configuration>
    <doclet>net.steppschuh.markdowngenerator.MarkdownDoclet</doclet>
    <docletArtifact>
      <groupId>de.neuhalje.doclet</groupId>
      <artifactId>markdown-doclet</artifactId>
      <version>LATEST</version>
    </docletArtifact>
    <destDir>docs/guides/api</destDir>
  </configuration>
</plugin>
```

Run: `mvn javadoc:javadoc -pl api`

Inspect output with the same criteria as Step 3. Run twice to verify determinism.

- [ ] **Step 5: Compare and decide**

Evaluate against criteria:
1. Output quality — readable, well-structured markdown
2. Deterministic — identical output for identical input
3. Type hierarchy — includes implements/extends
4. JDK compatibility
5. Integration simplicity

Document decision in a brief note. If neither tool produces adequate output, evaluate whether the standard javadoc doclet with custom post-processing is viable.

- [ ] **Step 6: Clean up**

Remove temporary pom.xml changes. Delete /tmp/doclet-eval/.

---

### Task 2: Evaluate TypeDoc

Evaluate typedoc + typedoc-plugin-markdown against casehub-pages.

**Files:**
- Read: `/Users/mdproctor/claude/casehub/pages/package.json`
- Read: pages source entry points
- Create: temporary typedoc.json in pages

**Interfaces:**
- Produces: confirmed TypeDoc configuration approach

- [ ] **Step 1: Install TypeDoc in pages**

```bash
npm install --save-dev typedoc typedoc-plugin-markdown --prefix /Users/mdproctor/claude/casehub/pages
```

- [ ] **Step 2: Create typedoc.json**

Create `/Users/mdproctor/claude/casehub/pages/typedoc.json`:

```json
{
  "$schema": "https://typedoc.org/schema.json",
  "entryPoints": ["src/index.ts"],
  "out": "docs/guides/api",
  "plugin": ["typedoc-plugin-markdown"],
  "outputFileStrategy": "members",
  "disableSources": true,
  "hideGenerator": true
}
```

`disableSources: true` prevents source file paths (which are machine-specific) from appearing in output. `hideGenerator: true` prevents the "Generated by TypeDoc" footer.

- [ ] **Step 3: Run TypeDoc**

```bash
npx typedoc --options typedoc.json --prefix /Users/mdproctor/claude/casehub/pages
```

Inspect output:
- Are interface members documented with types?
- Are generics resolved?
- Does it respect barrel exports (only exported types)?
- Run twice — is output identical?

- [ ] **Step 4: Check blocks-ui compatibility**

Verify blocks-ui's monorepo structure can be handled:

```bash
ls /Users/mdproctor/claude/casehub/blocks-ui/packages/*/src/index.ts 2>/dev/null
```

If workspaces exist, verify TypeDoc supports multiple entry points. Note any additional configuration needed for blocks-ui.

- [ ] **Step 5: Clean up**

Remove typedoc.json and node_modules changes if not proceeding immediately. Keep notes on configuration.

---

### Task 3: Configure Engine API Docs (Java Pilot)

Add doclet configuration to engine-api and generate the first API docs.

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/engine/api/pom.xml` (add doclet plugin)
- Create: `/Users/mdproctor/claude/casehub/engine/docs/guides/api/` (generated output)

**Interfaces:**
- Consumes: chosen doclet from Task 1
- Produces: engine's `docs/guides/api/*.md` files (consumed by sync-guides.sh)

- [ ] **Step 1: Add doclet plugin to engine-api pom.xml**

Add the chosen doclet as a maven-javadoc-plugin configuration in engine-api's `pom.xml`. Use the configuration determined in Task 1.

- [ ] **Step 2: Generate API docs**

```bash
# Delete existing output (clean slate)
rm -rf /Users/mdproctor/claude/casehub/engine/docs/guides/api/

# Run the doclet
mvn javadoc:javadoc -pl api -f /Users/mdproctor/claude/casehub/engine/pom.xml
```

- [ ] **Step 3: Verify output**

Check that:
- `docs/guides/api/` exists with markdown files
- Key types are present: `AgentRoutingStrategy.md`, `CaseDefinition.md`
- Each file has the `<!-- Generated — do not edit -->` header (add via post-processing if doclet doesn't include it)
- Type hierarchy information is present

- [ ] **Step 4: Verify determinism**

```bash
cp -r /Users/mdproctor/claude/casehub/engine/docs/guides/api/ /tmp/api-run1
rm -rf /Users/mdproctor/claude/casehub/engine/docs/guides/api/
mvn javadoc:javadoc -pl api -f /Users/mdproctor/claude/casehub/engine/pom.xml
diff -r /tmp/api-run1 /Users/mdproctor/claude/casehub/engine/docs/guides/api/
```

Expected: no differences. If differences exist, identify the non-deterministic elements and add post-processing to stabilise.

- [ ] **Step 5: Commit to engine repo**

```bash
git -C /Users/mdproctor/claude/casehub/engine add docs/guides/api/ api/pom.xml
git -C /Users/mdproctor/claude/casehub/engine commit -m "docs(#402): add generated API reference for engine-api [skip ci]"
```

---

### Task 4: Configure Pages API Docs (TypeScript Pilot)

Add TypeDoc configuration to pages and generate the first TS API docs.

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/pages/package.json` (add typedoc scripts + devDependencies)
- Create: `/Users/mdproctor/claude/casehub/pages/typedoc.json`
- Create: `/Users/mdproctor/claude/casehub/pages/docs/guides/api/` (generated output)

**Interfaces:**
- Consumes: TypeDoc configuration from Task 2
- Produces: pages' `docs/guides/api/*.md` files (consumed by sync-guides.sh)

- [ ] **Step 1: Add TypeDoc to package.json**

Add devDependencies and a generate script:

```json
{
  "devDependencies": {
    "typedoc": "^0.27",
    "typedoc-plugin-markdown": "^4.12"
  },
  "scripts": {
    "generate-api-docs": "rm -rf docs/guides/api && typedoc"
  }
}
```

- [ ] **Step 2: Create typedoc.json**

Use the configuration validated in Task 2.

- [ ] **Step 3: Generate and verify**

```bash
npm run generate-api-docs --prefix /Users/mdproctor/claude/casehub/pages
```

Verify output structure, type coverage, and determinism (same as Task 2 Steps 3-4).

- [ ] **Step 4: Add generated header**

If TypeDoc doesn't include `<!-- Generated — do not edit -->` headers, add a post-generation script step:

```bash
# Add header to each generated .md file
find docs/guides/api -name "*.md" -exec sed -i '' '1i\
<!-- Generated — do not edit -->\
' {} +
```

- [ ] **Step 5: Commit to pages repo**

```bash
git -C /Users/mdproctor/claude/casehub/pages add docs/guides/api/ typedoc.json package.json
git -C /Users/mdproctor/claude/casehub/pages commit -m "docs(#402): add generated API reference for pages [skip ci]"
```

---

### Task 5: Update sync-guides.sh Gate + Verify Aggregation

Update the gate condition and verify parent aggregates the new api/ subdirectory.

**Files:**
- Modify: `/Users/mdproctor/claude/casehub/parent/scripts/sync-guides.sh` (gate condition)

**Interfaces:**
- Consumes: engine and pages `docs/guides/api/` from Tasks 3-4
- Produces: aggregated `docs/repos/casehub-engine/api/` and `docs/repos/casehub-pages/api/` in parent

- [ ] **Step 1: Update sync-guides.sh gate condition**

Change the gate (around line 63) from:

```bash
if [ ! -f "$clone_dir/docs/guides/consumer-guide.md" ] && [ ! -f "$clone_dir/docs/guides/contributor-guide.md" ]; then
```

to:

```bash
if [ ! -f "$clone_dir/docs/guides/consumer-guide.md" ] && [ ! -f "$clone_dir/docs/guides/contributor-guide.md" ] && [ ! -d "$clone_dir/docs/guides/api" ]; then
```

- [ ] **Step 2: Run sync for engine**

```bash
bash /Users/mdproctor/claude/casehub/parent/scripts/sync-guides.sh --local --repo engine
```

Expected: `UPDATED engine → docs/repos/casehub-engine/`

- [ ] **Step 3: Verify api/ arrived**

```bash
ls /Users/mdproctor/claude/casehub/parent/docs/repos/casehub-engine/api/
```

Expected: markdown files matching what engine generated.

- [ ] **Step 4: Run sync for pages**

```bash
bash /Users/mdproctor/claude/casehub/parent/scripts/sync-guides.sh --local --repo pages
```

- [ ] **Step 5: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add scripts/sync-guides.sh docs/repos/
git -C /Users/mdproctor/claude/casehub/parent commit -m "feat(#402): update sync-guides gate + aggregate engine and pages API docs"
```

---

### Task 6: Build Cross-Repo Overlay Script

Write the Python script that reads aggregated doclet output and generates `cross-repo-implementations.md`.

**Files:**
- Create: `/Users/mdproctor/claude/casehub/parent/scripts/api-catalogue/generate_overlay.py`
- Create: `/Users/mdproctor/claude/casehub/parent/docs/api/cross-repo-implementations.md` (generated output)
- Test: `/Users/mdproctor/claude/casehub/parent/scripts/tests/test_generate_overlay.py`

**Interfaces:**
- Consumes: aggregated `docs/repos/*/api/*.md` files
- Produces: `docs/api/cross-repo-implementations.md`

- [ ] **Step 1: Write the test**

Create `/Users/mdproctor/claude/casehub/parent/scripts/tests/test_generate_overlay.py`:

```python
import tempfile
import os
from pathlib import Path

def create_test_docs(base_dir):
    """Create minimal doclet output for testing."""
    engine_dir = Path(base_dir) / "docs/repos/casehub-engine/api"
    engine_dir.mkdir(parents=True)

    blocks_dir = Path(base_dir) / "docs/repos/casehub-blocks/api"
    blocks_dir.mkdir(parents=True)

    # Engine defines the interface
    (engine_dir / "AgentRoutingStrategy.md").write_text(
        "<!-- Generated — do not edit -->\n"
        "# AgentRoutingStrategy\n\n"
        "> `io.casehub.api.spi.routing` · casehub-engine-api\n\n"
        "## Implements / Extends\n\n"
        "- Interface\n"
    )

    # Blocks has two implementations
    (blocks_dir / "LlmAgentRoutingStrategy.md").write_text(
        "<!-- Generated — do not edit -->\n"
        "# LlmAgentRoutingStrategy\n\n"
        "> `io.casehub.blocks.routing.agent` · casehub-blocks\n\n"
        "## Implements / Extends\n\n"
        "- Implements `AgentRoutingStrategy`\n"
    )
    (blocks_dir / "CbrAgentRoutingStrategy.md").write_text(
        "<!-- Generated — do not edit -->\n"
        "# CbrAgentRoutingStrategy\n\n"
        "> `io.casehub.blocks.routing.agent` · casehub-blocks\n\n"
        "## Implements / Extends\n\n"
        "- Implements `AgentRoutingStrategy`\n"
    )

    return base_dir


def test_discovers_implementations():
    with tempfile.TemporaryDirectory() as tmpdir:
        create_test_docs(tmpdir)
        # Import will be adjusted once module structure is known
        from generate_overlay import scan_implementations
        result = scan_implementations(Path(tmpdir) / "docs/repos")
        assert "AgentRoutingStrategy" in result
        impls = result["AgentRoutingStrategy"]
        assert len(impls) == 2
        assert any(i["class"] == "LlmAgentRoutingStrategy" for i in impls)
        assert any(i["class"] == "CbrAgentRoutingStrategy" for i in impls)


def test_generates_markdown():
    with tempfile.TemporaryDirectory() as tmpdir:
        create_test_docs(tmpdir)
        from generate_overlay import scan_implementations, render_markdown
        impls = scan_implementations(Path(tmpdir) / "docs/repos")
        md = render_markdown(impls)
        assert "# Cross-Repo SPI Implementations" in md
        assert "AgentRoutingStrategy" in md
        assert "LlmAgentRoutingStrategy" in md
        assert "casehub-blocks" in md


def test_ignores_interfaces_with_no_cross_repo_impls():
    with tempfile.TemporaryDirectory() as tmpdir:
        create_test_docs(tmpdir)
        from generate_overlay import scan_implementations, render_markdown
        impls = scan_implementations(Path(tmpdir) / "docs/repos")
        md = render_markdown(impls)
        # Interfaces only implemented within their own repo shouldn't appear
        # (AgentRoutingStrategy has cross-repo impls so it should appear)
        assert "AgentRoutingStrategy" in md
```

- [ ] **Step 2: Run tests to verify they fail**

```bash
python3 -m pytest /Users/mdproctor/claude/casehub/parent/scripts/tests/test_generate_overlay.py -v
```

Expected: FAIL — `generate_overlay` module not found.

- [ ] **Step 3: Write the overlay script**

Create `/Users/mdproctor/claude/casehub/parent/scripts/api-catalogue/generate_overlay.py`:

```python
#!/usr/bin/env python3
"""
Scan aggregated API docs and generate cross-repo SPI implementation matrix.

Reads docs/repos/*/api/*.md for "Implements / Extends" sections.
Produces docs/api/cross-repo-implementations.md.
"""
import re
import sys
from pathlib import Path
from collections import defaultdict


def scan_implementations(repos_dir: Path) -> dict[str, list[dict]]:
    """Scan all repo api docs for implements/extends declarations."""
    implementations = defaultdict(list)

    for repo_dir in sorted(repos_dir.iterdir()):
        if not repo_dir.is_dir():
            continue
        api_dir = repo_dir / "api"
        if not api_dir.is_dir():
            continue

        repo_name = repo_dir.name

        for md_file in sorted(api_dir.glob("*.md")):
            if md_file.name in ("index.md", "INDEX.md"):
                continue

            content = md_file.read_text()
            class_name = md_file.stem

            in_hierarchy = False
            for line in content.splitlines():
                if line.strip().startswith("## Implements") or line.strip().startswith("## Extends"):
                    in_hierarchy = True
                    continue
                if in_hierarchy and line.startswith("## "):
                    break
                if in_hierarchy and "Implements" in line:
                    match = re.search(r'`(\w+)`', line)
                    if match:
                        interface_name = match.group(1)
                        implementations[interface_name].append({
                            "repo": repo_name,
                            "class": class_name,
                        })

    return dict(implementations)


def render_markdown(implementations: dict[str, list[dict]]) -> str:
    """Render the cross-repo implementation matrix as markdown."""
    lines = [
        "<!-- Generated — do not edit -->",
        "# Cross-Repo SPI Implementations",
        "",
        "SPIs with implementations across multiple repos.",
        "",
    ]

    for interface_name in sorted(implementations.keys()):
        impls = implementations[interface_name]
        repos = {i["repo"] for i in impls}
        if len(repos) < 2:
            continue

        lines.append(f"## {interface_name}")
        lines.append("")
        lines.append("| Repo | Implementation |")
        lines.append("|------|---------------|")
        for impl in sorted(impls, key=lambda i: (i["repo"], i["class"])):
            lines.append(f"| {impl['repo']} | `{impl['class']}` |")
        lines.append("")

    return "\n".join(lines) + "\n"


def main():
    if len(sys.argv) < 2:
        print("Usage: generate_overlay.py <parent-repo-root>")
        sys.exit(1)

    root = Path(sys.argv[1])
    repos_dir = root / "docs" / "repos"

    if not repos_dir.is_dir():
        print(f"Error: {repos_dir} not found")
        sys.exit(1)

    implementations = scan_implementations(repos_dir)
    markdown = render_markdown(implementations)

    output_dir = root / "docs" / "api"
    output_dir.mkdir(parents=True, exist_ok=True)
    output_file = output_dir / "cross-repo-implementations.md"
    output_file.write_text(markdown)

    print(f"Generated {output_file} ({len(implementations)} SPIs found)")


if __name__ == "__main__":
    main()
```

- [ ] **Step 4: Run tests to verify they pass**

```bash
PYTHONPATH=/Users/mdproctor/claude/casehub/parent/scripts/api-catalogue python3 -m pytest /Users/mdproctor/claude/casehub/parent/scripts/tests/test_generate_overlay.py -v
```

Expected: all tests PASS.

- [ ] **Step 5: Run against real data**

```bash
python3 /Users/mdproctor/claude/casehub/parent/scripts/api-catalogue/generate_overlay.py /Users/mdproctor/claude/casehub/parent
```

Inspect `docs/api/cross-repo-implementations.md`. Verify it found real implementations.

- [ ] **Step 6: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add scripts/api-catalogue/ scripts/tests/test_generate_overlay.py docs/api/
git -C /Users/mdproctor/claude/casehub/parent commit -m "feat(#402): cross-repo SPI implementation overlay script + tests"
```

---

### Task 7: Write Platform API Index + Update Platform Docs

Create the platform-wide API discovery index and update existing indexes to link to it.

**Files:**
- Create: `/Users/mdproctor/claude/casehub/parent/docs/api/INDEX.md`
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/INDEX.md` (add link)
- Modify: `/Users/mdproctor/claude/casehub/parent/docs/consumer-index.md` (add link)

**Interfaces:**
- Consumes: aggregated `docs/repos/*/api/` directories

- [ ] **Step 1: Write docs/api/INDEX.md**

```markdown
# CaseHub API Reference

> Machine-generated API documentation for LLM and RAG consumption.
> Each repo's API reference is generated from source by javadoc doclets
> (Java) or TypeDoc (TypeScript). This index links to all available
> API references.

## Per-Repo API References

| Repo | API Surface | Reference |
|------|------------|-----------|
| **engine** | CaseDefinition, AgentRoutingStrategy, bindings, goals, planning SPIs | [casehub-engine API](../repos/casehub-engine/api/index.md) |
| **pages** | ConfigurablePanel, DataReceiver, form components, data pipelines | [casehub-pages API](../repos/casehub-pages/api/index.md) |

*(Additional repos will be added as they are configured)*

## Cross-Repo SPI Implementations

[cross-repo-implementations.md](cross-repo-implementations.md) — which repos
implement which platform SPIs. Updated automatically when repos are synced.

## Relationship to Other Docs

- **Consumer guides** (`docs/repos/*/consumer-guide.md`) — explain *when* and *why* to use each API
- **This reference** (`docs/repos/*/api/`) — documents *what* the exact signatures and types are
- **Cross-repo implementations** — shows *who else* implements each SPI
```

- [ ] **Step 2: Add link to docs/INDEX.md**

Add under the "### Guides & Tools" section:

```markdown
- [API Reference](api/INDEX.md) — machine-generated type signatures, method contracts (per-repo + cross-repo SPI matrix)
```

- [ ] **Step 3: Add link to consumer-index.md**

Add a brief note at the top after the intro paragraph:

```markdown
> For exact type signatures and method contracts, see the [API Reference](api/INDEX.md).
```

- [ ] **Step 4: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add docs/api/INDEX.md docs/INDEX.md docs/consumer-index.md
git -C /Users/mdproctor/claude/casehub/parent commit -m "docs(#402): platform API index + links from INDEX.md and consumer-index.md"
```

---

### Task 8: Parent CI Workflow

Create the GitHub Actions workflow that runs the aggregation and overlay.

**Files:**
- Create: `/Users/mdproctor/claude/casehub/parent/.github/workflows/generate-api-catalogue.yml`

**Interfaces:**
- Consumes: sync-guides.sh, generate_overlay.py
- Produces: automated commits to parent main when API docs change

- [ ] **Step 1: Write the workflow**

```yaml
name: Generate API Catalogue

on:
  push:
    branches: [main]
    paths:
      - 'docs/repos/*/api/**'
  schedule:
    - cron: '0 6 * * 1'  # Weekly Monday 6am UTC
  workflow_dispatch:

permissions:
  contents: write

jobs:
  generate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
          token: ${{ secrets.GH_PAT }}

      - name: Run cross-repo overlay
        run: python3 scripts/api-catalogue/generate_overlay.py .

      - name: Check for changes
        id: diff
        run: |
          git diff --quiet docs/api/ && echo "changed=false" >> $GITHUB_OUTPUT || echo "changed=true" >> $GITHUB_OUTPUT

      - name: Commit and push
        if: steps.diff.outputs.changed == 'true'
        run: |
          git config user.name "github-actions[bot]"
          git config user.email "github-actions[bot]@users.noreply.github.com"
          git add docs/api/
          git commit -m "docs(#402): regenerate API catalogue [skip ci]"
          git push
```

- [ ] **Step 2: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/parent add .github/workflows/generate-api-catalogue.yml
git -C /Users/mdproctor/claude/casehub/parent commit -m "ci(#402): add API catalogue generation workflow"
```

---

## Deferred Work (not in this plan)

These items are mechanical rollout once the pipeline works end-to-end:

- **Roll out to remaining Java repos** — add doclet config to work, eidos, qhorus, ledger, worker, platform, blocks, neocortex, ras, desiredstate, connectors, ops, iot. Same pattern as engine (Task 3).
- **Roll out to remaining TS repos** — add TypeDoc config to blocks-ui. Same pattern as pages (Task 4).
- **Per-repo CI workflows** — add generation step to each repo's existing workflow. Template from Task 8.
- **Update docs/api/INDEX.md** — add rows for each newly configured repo.
