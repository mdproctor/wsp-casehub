# CaseHub Org Website Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a single landing page at `/Users/mdproctor/claude/casehub/casehubio.github.io/` serving as the casehubio GitHub org website, communicating "CaseHub — An AI Fusion Harness" with project cards, an AI Fusion explainer, and an SVG architecture layer diagram.

**Architecture:** Plain HTML + CSS (no build step, no Jekyll). A single `index.html` at repo root with `assets/css/main.css` — paths are Jekyll-migration-friendly. Theme and logo are copied from `casehub-poc/docs/assets/css/main.css`.

**Tech Stack:** HTML5, CSS3 (custom properties, CSS Grid, inline SVG). No JavaScript. No build tools. GitHub Pages serves directly from repo root.

**Spec:** `/Users/mdproctor/claude/casehub/parent/docs/superpowers/specs/2026-05-23-casehubio-website-design.md`

---

## File Map

| File | Action | Responsibility |
|------|--------|----------------|
| `assets/css/main.css` | Create (copy + extend) | Theme vars, nav, hero, AI Fusion section, architecture diagram, project cards, footer |
| `index.html` | Create | Complete single-page HTML — all sections |
| `README.md` | Create | Local dev instructions + Jekyll migration notes |

---

### Task 1: Initialize repo and CSS

**Files:**
- Create: `/Users/mdproctor/claude/casehub/casehubio.github.io/assets/css/main.css`

- [ ] **Step 1: Create the directory structure and initialize git**

```bash
mkdir -p /Users/mdproctor/claude/casehub/casehubio.github.io/assets/css
git -C /Users/mdproctor/claude/casehub/casehubio.github.io init
```

- [ ] **Step 2: Copy the poc CSS as a base**

```bash
cp /Users/mdproctor/claude/casehub-poc/docs/assets/css/main.css \
   /Users/mdproctor/claude/casehub/casehubio.github.io/assets/css/main.css
```

- [ ] **Step 3: Append new styles to main.css**

Open `/Users/mdproctor/claude/casehub/casehubio.github.io/assets/css/main.css` and append the following after the existing `/* ── Doc Placeholder ── */` block at the end of the file:

```css
/* ── Hero Tagline ──────────────────────────────────────── */
.hero-tagline {
  color: var(--accent);
  font-size: clamp(16px, 2.5vw, 24px);
  font-weight: 600;
  margin-bottom: 16px;
  letter-spacing: 0.5px;
}

/* ── What is AI Fusion ─────────────────────────────────── */
.what-is {
  max-width: 760px;
  margin: 0 auto;
  padding: 72px 48px;
  text-align: center;
}
.what-is h2 {
  font-size: 22px;
  font-weight: 600;
  color: var(--text);
  margin-bottom: 20px;
}
.what-is .fusion-desc {
  color: var(--text-muted);
  font-size: 15px;
  line-height: 1.85;
  margin-bottom: 32px;
}
.fusion-tags {
  display: flex;
  gap: 10px;
  justify-content: center;
  flex-wrap: wrap;
}
.fusion-tag {
  border: 1px solid var(--border);
  padding: 5px 14px;
  border-radius: 3px;
  font-size: 11px;
  color: var(--text-muted);
  text-transform: uppercase;
  letter-spacing: 1px;
}
.fusion-tag.classical { border-color: var(--accent); color: var(--accent); }
.fusion-tag.llm { border-color: #7ecfdf; color: #7ecfdf; }
.fusion-tag.sep { border: none; color: var(--text-muted); font-size: 16px; padding: 0 4px; }

/* ── Architecture Diagram ──────────────────────────────── */
.arch-section {
  background: var(--bg-card);
  border-top: 1px solid var(--border);
  border-bottom: 1px solid var(--border);
  padding: 56px 48px;
}
.arch-section h2 {
  text-align: center;
  font-size: 20px;
  font-weight: 600;
  color: var(--text);
  margin-bottom: 36px;
  letter-spacing: 0.5px;
}
.arch-diagram {
  max-width: 860px;
  margin: 0 auto;
}
.arch-diagram svg {
  width: 100%;
  height: auto;
  display: block;
}

/* ── Project Cards ─────────────────────────────────────── */
.projects-wrap {
  padding: 64px 48px;
  max-width: 1100px;
  margin: 0 auto;
}
.apps-bg {
  background: var(--bg-card);
  border-top: 1px solid var(--border);
}
.apps-bg .projects-wrap {
  max-width: 1100px;
  margin: 0 auto;
}
.projects-header {
  margin-bottom: 36px;
}
.projects-header h2 {
  font-size: 22px;
  font-weight: 700;
  color: var(--text);
  margin-bottom: 4px;
}
.projects-header p {
  color: var(--text-muted);
  font-size: 13px;
}
.project-cards {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 20px;
}
@media (max-width: 900px) {
  .project-cards { grid-template-columns: repeat(2, 1fr); }
}
@media (max-width: 580px) {
  .project-cards { grid-template-columns: 1fr; }
  .projects-wrap { padding: 40px 24px; }
  .what-is { padding: 48px 24px; }
  .arch-section { padding: 40px 24px; }
}
.project-card {
  background: var(--bg-card);
  border: 1px solid var(--border);
  border-radius: 6px;
  padding: 24px;
  display: flex;
  flex-direction: column;
  gap: 10px;
  transition: border-color 0.2s;
}
.apps-bg .project-card {
  background: var(--bg-deep);
}
.project-card:hover { border-color: var(--accent); }
.card-repo {
  font-family: var(--mono);
  font-size: 11px;
  color: var(--accent);
  background: rgba(42,168,196,0.07);
  display: inline-block;
  padding: 3px 8px;
  border-radius: 3px;
  border: 1px solid rgba(42,168,196,0.18);
  width: fit-content;
}
.card-headline {
  font-size: 13px;
  font-weight: 600;
  color: var(--text);
  line-height: 1.45;
}
.card-desc {
  font-size: 12px;
  color: var(--text-muted);
  line-height: 1.75;
  flex: 1;
}
.card-link {
  font-size: 12px;
  color: var(--accent);
  margin-top: 4px;
  width: fit-content;
}
.card-link:hover { text-decoration: underline; }
```

- [ ] **Step 4: Commit the CSS**

```bash
git -C /Users/mdproctor/claude/casehub/casehubio.github.io add assets/css/main.css
git -C /Users/mdproctor/claude/casehub/casehubio.github.io commit -m "feat: add site CSS — poc theme extended with card and diagram styles"
```

---

### Task 2: Complete index.html

**Files:**
- Create: `/Users/mdproctor/claude/casehub/casehubio.github.io/index.html`

- [ ] **Step 1: Create index.html with the complete page**

Create `/Users/mdproctor/claude/casehub/casehubio.github.io/index.html` with this full content:

```html
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>CaseHub — An AI Fusion Harness</title>
  <meta name="description" content="CaseHub is a production-grade AI Fusion Harness fusing Classical AI and LLM AI for regulated multi-agent systems, built on Quarkus.">
  <link rel="stylesheet" href="/assets/css/main.css">
</head>
<body class="has-hero">

  <!-- Nav -->
  <nav class="site-nav">
    <a class="nav-logo" href="/">
      <svg width="22" height="22" viewBox="0 0 24 24" fill="none" xmlns="http://www.w3.org/2000/svg" aria-hidden="true">
        <rect x="2" y="2" width="9" height="9" rx="1.5" fill="#2aa8c4" opacity="0.9"/>
        <rect x="13" y="2" width="9" height="9" rx="1.5" fill="#2aa8c4" opacity="0.6"/>
        <rect x="2" y="13" width="9" height="9" rx="1.5" fill="#2aa8c4" opacity="0.6"/>
        <rect x="13" y="13" width="9" height="9" rx="1.5" fill="#2aa8c4" opacity="0.3"/>
      </svg>
      <span>CaseHub</span>
    </a>
    <div class="nav-links">
      <a href="https://github.com/casehubio" class="nav-link" target="_blank" rel="noopener">GitHub &#x2197;</a>
    </div>
  </nav>

  <!-- Hero -->
  <section class="hero">
    <div class="hero-grid"></div>
    <div class="hero-content">
      <p class="hero-eyebrow">Coming Soon</p>
      <h1 class="hero-title">CaseHub</h1>
      <p class="hero-tagline">An AI Fusion Harness</p>
      <p class="hero-sub">Where Classical AI meets LLM AI — production-grade orchestration for regulated multi-agent systems, built on Quarkus.</p>
      <div class="hero-btns">
        <a href="https://github.com/casehubio" class="btn-primary" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>
    </div>
  </section>

  <!-- What is AI Fusion -->
  <section class="what-is">
    <h2>What is AI Fusion?</h2>
    <p class="fusion-desc">Classical AI brings structure — rules engines, formal process models (CMMN), Blackboard Architecture, and deterministic reasoning. LLM AI brings adaptability — autonomous agents, natural language understanding, and emergent problem-solving. CaseHub fuses both: a harness where each kind of intelligence does what it does best, coordinated by a compliance-first orchestration layer.</p>
    <div class="fusion-tags">
      <span class="fusion-tag classical">Blackboard Architecture</span>
      <span class="fusion-tag classical">CMMN Semantics</span>
      <span class="fusion-tag classical">Rules Engines</span>
      <span class="fusion-tag sep">+</span>
      <span class="fusion-tag llm">LLM Agents</span>
      <span class="fusion-tag llm">Agent Mesh</span>
      <span class="fusion-tag llm">Adaptive Reasoning</span>
    </div>
  </section>

  <!-- Architecture Diagram -->
  <section class="arch-section">
    <h2>Platform Architecture</h2>
    <div class="arch-diagram">
      <!--
        Layer diagram: 4 horizontal bands, bottom-to-top.
        SVG y increases downward, so Applications is at top (y=20),
        Foundation is at bottom (y=240). Arrows point upward showing
        each layer provides capabilities to the layer above.
        Annotation brackets on right: Classical AI spans
        Orchestration+Foundation; LLM AI spans Runtime+Foundation.
        Foundation has accent border — it is the AI Fusion Core.
      -->
      <svg viewBox="0 0 860 340" xmlns="http://www.w3.org/2000/svg"
           role="img"
           aria-label="CaseHub platform architecture — four tiers from bottom: Foundation, Orchestration, Runtime, Applications">
        <defs>
          <!-- Upward-pointing chevron marker -->
          <marker id="arr" markerWidth="8" markerHeight="7" refX="4" refY="0" orient="auto">
            <path d="M0,7 L4,0 L8,7" fill="none" stroke="#1a2e38" stroke-width="1.2"/>
          </marker>
        </defs>

        <!-- ── Applications band (y=20, h=62) ── -->
        <rect x="20" y="20" width="620" height="62" rx="4"
              fill="#0e1820" stroke="#1a2e38" stroke-width="1"/>
        <text x="32" y="38"
              font-size="8" fill="#4a7a8a" letter-spacing="2.5"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif"
              font-weight="700">APPLICATIONS</text>
        <text x="32"  y="58" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">devtown</text>
        <text x="112" y="58" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">aml</text>
        <text x="158" y="58" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">clinical</text>
        <text x="242" y="58" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">quarkmind</text>

        <!-- Arrow in gap y=82–100 (Runtime → Applications) -->
        <line x1="320" y1="97" x2="320" y2="83" stroke="#1a2e38" stroke-width="1" marker-end="url(#arr)"/>

        <!-- ── Runtime band (y=100, h=52) ── -->
        <rect x="20" y="100" width="620" height="52" rx="4"
              fill="#0e1820" stroke="#1a2e38" stroke-width="1"/>
        <text x="32" y="118"
              font-size="8" fill="#4a7a8a" letter-spacing="2.5"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif"
              font-weight="700">RUNTIME</text>
        <text x="32" y="138" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">claudony</text>

        <!-- Arrow in gap y=152–170 (Orchestration → Runtime) -->
        <line x1="320" y1="167" x2="320" y2="153" stroke="#1a2e38" stroke-width="1" marker-end="url(#arr)"/>

        <!-- ── Orchestration band (y=170, h=52) ── -->
        <rect x="20" y="170" width="620" height="52" rx="4"
              fill="#0e1820" stroke="#1a2e38" stroke-width="1"/>
        <text x="32" y="188"
              font-size="8" fill="#4a7a8a" letter-spacing="2.5"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif"
              font-weight="700">ORCHESTRATION</text>
        <text x="32" y="208" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">casehub-engine</text>

        <!-- Arrow in gap y=222–240 (Foundation → Orchestration) -->
        <line x1="320" y1="237" x2="320" y2="223" stroke="#1a2e38" stroke-width="1" marker-end="url(#arr)"/>

        <!-- ── Foundation band (y=240, h=84) — accent border = AI Fusion Core ── -->
        <rect x="20" y="240" width="620" height="84" rx="4"
              fill="#0e1820" stroke="#2aa8c4" stroke-width="1.5"/>
        <text x="32" y="259"
              font-size="8" fill="#2aa8c4" letter-spacing="2.5"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif"
              font-weight="700">FOUNDATION</text>
        <text x="32"  y="282" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">platform</text>
        <text x="116" y="282" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">ledger</text>
        <text x="188" y="282" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">work</text>
        <text x="240" y="282" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">qhorus</text>
        <text x="320" y="282" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">connectors</text>
        <text x="440" y="282" font-size="11" fill="#b8d8e0" font-family="'JetBrains Mono','Fira Code','Courier New',monospace">eidos</text>
        <text x="510" y="310" font-size="9" fill="#2aa8c4" opacity="0.55"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif">AI Fusion Core</text>

        <!-- ── Classical AI annotation ── -->
        <!-- Bracket spans Orchestration+Foundation: y=170 to y=324 -->
        <path d="M652,170 L664,170 L664,324 L652,324"
              stroke="#4a7a8a" stroke-width="1" fill="none"/>
        <text x="672" y="242"
              font-size="10" fill="#4a7a8a" font-weight="600"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif">Classical AI</text>
        <text x="672" y="257"
              font-size="9" fill="#2d5060"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif">Blackboard · CMMN</text>

        <!-- ── LLM AI annotation ── -->
        <!-- Bracket spans Runtime+Foundation: y=100 to y=324 -->
        <path d="M742,100 L754,100 L754,324 L742,324"
              stroke="#2aa8c4" stroke-width="1" fill="none" opacity="0.45"/>
        <text x="762" y="202"
              font-size="10" fill="#2aa8c4" font-weight="600" opacity="0.8"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif">LLM AI</text>
        <text x="762" y="217"
              font-size="9" fill="#1a5a6a"
              font-family="-apple-system,BlinkMacSystemFont,'Segoe UI',sans-serif">Agents · Mesh</text>
      </svg>
    </div>
  </section>

  <!-- Foundation Cards -->
  <div class="projects-wrap">
    <div class="projects-header">
      <h2>Foundation</h2>
      <p>The platform beneath the harness</p>
    </div>
    <div class="project-cards">

      <div class="project-card">
        <span class="card-repo">casehub-platform</span>
        <span class="card-headline">Zero-dependency SPI layer — Path, Preferences, Identity</span>
        <p class="card-desc">casehub-platform defines thin, domain-agnostic SPIs (Path, Preferences, CurrentPrincipal) that every module in the stack can implement without framework dependencies. The three-tier model keeps platform-api pure Java while optional config and OIDC layers integrate cleanly with Quarkus via CDI.</p>
        <a href="https://github.com/casehubio/platform" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

      <div class="project-card">
        <span class="card-repo">casehub-ledger</span>
        <span class="card-headline">Immutable, cryptographically tamper-evident audit ledger with trust scoring</span>
        <p class="card-desc">The ledger provides immutable entries, attestations, and Bayesian trust scores with Merkle frontier operations for verifiable case and actor accountability. It enables GDPR Art.17 erasure, trust bootstrapping, and W3C PROV-DM lineage export without knowledge of business domain.</p>
        <a href="https://github.com/casehubio/ledger" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

      <div class="project-card">
        <span class="card-repo">casehub-work</span>
        <span class="card-headline">Human task lifecycle — inbox, SLA, delegation, routing, and audit trail</span>
        <p class="card-desc">WorkItem coordinates humans via formal SLA policy, delegation, escalation, spawn semantics, and skill profiling with optional ledger attachment and distributed queue views. Usable standalone or integrated with CaseHub and Qhorus for multi-agent coordination.</p>
        <a href="https://github.com/casehubio/work" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

      <div class="project-card">
        <span class="card-repo">casehub-qhorus</span>
        <span class="card-headline">Agent communication mesh with formal speech-act accountability</span>
        <p class="card-desc">Qhorus models every agent interaction as a typed speech act with commitments, shared artefacts, and channels — grounded in speech act theory and deontic logic. All writes flow through a single dispatch gate enforcing ACL, rate limiting, and ledger recording without bypass paths.</p>
        <a href="https://github.com/casehubio/qhorus" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

      <div class="project-card">
        <span class="card-repo">casehub-connectors</span>
        <span class="card-headline">Outbound message connectors — Slack, Teams, SMS, email</span>
        <p class="card-desc">Provides built-in implementations for Slack, Teams, Twilio SMS, WhatsApp, and email using pure HttpClient — no Camel, no vendor SDKs. This is the canonical outbound notification infrastructure; any module needing alerts or escalations uses this SPI rather than implementing its own.</p>
        <a href="https://github.com/casehubio/connectors" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

      <div class="project-card">
        <span class="card-repo">casehub-eidos</span>
        <span class="card-headline">Structured agent identity, capability discovery, and system prompt generation</span>
        <p class="card-desc">Eidos enables agents to register with identity, slot, capabilities, and disposition — then discover agents by slot or capability and render system prompts from descriptors. Any Quarkus app depending on casehub-eidos gains structured agent identity without touching LLM implementation.</p>
        <a href="https://github.com/casehubio/eidos" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

      <div class="project-card">
        <span class="card-repo">casehub-engine</span>
        <span class="card-headline">Hybrid choreography + Blackboard orchestration engine (CMMN semantics)</span>
        <p class="card-desc">The engine coordinates workers (AI agents, humans) via declarative case definitions, binding rules, and optional synchronous orchestration — supporting adaptive routing, synchronous tasks, and ledger attachment. Cases are defined in YAML DSL and persist via pluggable repositories.</p>
        <a href="https://github.com/casehubio/engine" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

      <div class="project-card">
        <span class="card-repo">claudony</span>
        <span class="card-headline">Remote Claude CLI sessions and unified ecosystem dashboard</span>
        <p class="card-desc">Claudony owns session lifecycle management, wires CaseHub + Qhorus together, and surfaces them in a browser/PWA dashboard with WebSocket streaming. It operates in two modes: server (owns sessions) and agent (MCP endpoint for a controller Claude instance).</p>
        <a href="https://github.com/casehubio/claudony" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
      </div>

    </div>
  </div>

  <!-- Application Cards -->
  <div class="apps-bg">
    <div class="projects-wrap">
      <div class="projects-header">
        <h2>Applications</h2>
        <p>Domain-specific harnesses built on the platform</p>
      </div>
      <div class="project-cards">

        <div class="project-card">
          <span class="card-repo">casehub-devtown</span>
          <span class="card-headline">AI-assisted PR review with adaptive specialist routing and tamper-evident records</span>
          <p class="card-desc">Casehub-devtown coordinates security, architecture, and test-coverage reviewers with SLA gates and content-driven adaptive routing — producing tamper-evident review records where every missed finding is traceable to its reviewer. Demonstrates Engine Layer 5 with binding-condition routing.</p>
          <a href="https://github.com/casehubio/devtown" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
        </div>

        <div class="project-card">
          <span class="card-repo">casehub-aml</span>
          <span class="card-headline">AML investigation — FinCEN-compliant, adaptive investigation paths</span>
          <p class="card-desc">Casehub-aml produces FinCEN-compliant, independently verifiable audit trails by coordinating entity resolution, pattern analysis, and OSINT screening agents with compliance officer human task gates. Demonstrates foundational CaseHub modules in a real financial-crime domain.</p>
          <a href="https://github.com/casehubio/aml" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
        </div>

        <div class="project-card">
          <span class="card-repo">casehub-clinical</span>
          <span class="card-headline">Clinical trial coordination — GCP/FDA/GDPR-compliant multi-site case management</span>
          <p class="card-desc">Casehub-clinical produces FDA-compliant, GDPR-aware audit trails across multi-site trial coordination with adaptive escalation for adverse events and protocol deviations. Demonstrates that GCP, FDA, and EMA requirements are structurally satisfied by the foundation, not by LLM coordination.</p>
          <a href="https://github.com/casehubio/clinical" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
        </div>

        <div class="project-card">
          <span class="card-repo">quarkmind</span>
          <span class="card-headline">StarCraft II game AI — proving the harness at millisecond granularity</span>
          <p class="card-desc">QuarkMind is a living lab demonstrating that the CaseHub agentic harness holds outside regulated enterprise domains at millisecond-granularity real-time coordination. Same foundation coordinates clinical trial monitors, AML investigators, code reviewers, and game AI plugins.</p>
          <a href="https://github.com/mdproctor/quarkmind" class="card-link" target="_blank" rel="noopener">View on GitHub &#x2197;</a>
        </div>

      </div>
    </div>
  </div>

  <!-- Footer -->
  <footer class="site-footer">
    <div class="footer-inner">
      <span class="footer-logo">CaseHub</span>
      <div class="footer-links">
        <a href="https://github.com/casehubio" target="_blank" rel="noopener">GitHub &#x2197;</a>
      </div>
      <span class="footer-tagline">Apache 2.0</span>
    </div>
  </footer>

</body>
</html>
```

- [ ] **Step 2: Open in browser and do a full visual review**

```bash
open /Users/mdproctor/claude/casehub/casehubio.github.io/index.html
```

Verify each section:
- **Nav**: CaseHub logo left, "GitHub ↗" right, absolute-positioned over hero
- **Hero**: dark grid background with teal glow, "Coming Soon" eyebrow (teal, uppercase), "CaseHub" large white h1, "An AI Fusion Harness" in teal below, muted subtitle, "View on GitHub ↗" teal button
- **AI Fusion**: centered section, "What is AI Fusion?" heading, muted paragraph, tags row with teal Classical AI tags + lighter-teal LLM AI tags
- **Architecture SVG**: dark card background, 4 bands stacked, Foundation has teal border, "AI Fusion Core" faint label in Foundation, upward arrow between each band, Classical AI bracket on right (muted), LLM AI bracket further right (accent, faint)
- **Foundation cards**: 8 cards in 3-col grid, dark card background (`bg-card`), repo name as small teal monospace badge, bold headline, muted 2-sentence description, "View on GitHub ↗" link, teal border on hover
- **Applications**: slightly different background (`bg-deep` cards on `bg-card` section background), 4 cards in 3-col grid (last row has 1 card)
- **Footer**: CaseHub wordmark left, GitHub link centre, "Apache 2.0" right

If the SVG annotation text at x=672+ is clipped on narrow screens, the `max-width: 860px` on `.arch-diagram` keeps it within bounds. The SVG scales down with `width: 100%; height: auto`.

- [ ] **Step 3: Commit**

```bash
git -C /Users/mdproctor/claude/casehub/casehubio.github.io add index.html
git -C /Users/mdproctor/claude/casehub/casehubio.github.io commit -m "feat: complete landing page — hero, AI Fusion, architecture SVG, project cards, footer"
```

---

### Task 3: README and remote setup

**Files:**
- Create: `/Users/mdproctor/claude/casehub/casehubio.github.io/README.md`

- [ ] **Step 1: Create README.md**

Create `/Users/mdproctor/claude/casehub/casehubio.github.io/README.md`:

```markdown
# casehubio.github.io

Org landing page for [casehubio](https://github.com/casehubio), served at https://casehubio.github.io.

## Stack

Plain HTML + CSS. No build step. GitHub Pages serves from repo root on `main`.

## Local development

Open `index.html` directly in a browser — no server required for basic review.

For accurate absolute path resolution (`/assets/css/main.css`), serve with Python:

```bash
python3 -m http.server 8080
# open http://localhost:8080
```

## Migration to Jekyll (planned)

File paths are already Jekyll-compatible. To migrate:

1. Add `_config.yml` with `title`, `url`, `baseurl: ""`
2. Extract `index.html` body into `_layouts/landing.html`
3. Replace `index.html` with Jekyll front matter pointing to `landing` layout
4. Add `Gemfile` with `jekyll` and `jekyll-feed`

## Deployment

Push to `main` on `casehubio/casehubio.github.io`. GitHub Pages deploys automatically (no workflow needed — org sites deploy from root of default branch).
```

- [ ] **Step 2: Commit README**

```bash
git -C /Users/mdproctor/claude/casehub/casehubio.github.io add README.md
git -C /Users/mdproctor/claude/casehub/casehubio.github.io commit -m "docs: add README with local dev and Jekyll migration notes"
```

- [ ] **Step 3: Add remote and push (once casehubio/casehubio.github.io exists on GitHub)**

The repo `casehubio/casehubio.github.io` must exist on GitHub before this step.
Create it at https://github.com/organizations/casehubio/repositories/new — name: `casehubio.github.io`, no template, no auto-init.

```bash
git -C /Users/mdproctor/claude/casehub/casehubio.github.io remote add origin https://github.com/casehubio/casehubio.github.io.git
git -C /Users/mdproctor/claude/casehub/casehubio.github.io branch -M main
git -C /Users/mdproctor/claude/casehub/casehubio.github.io push -u origin main
```

GitHub Pages activates automatically for org sites — `https://casehubio.github.io` serves from `main` branch root once pushed. No Pages configuration needed.

---

## Self-Review Checklist

- [x] **Spec coverage**: all spec sections covered — nav (GitHub-only), hero (Coming Soon + tagline + CTA), AI Fusion explainer, SVG diagram (4 bands + annotations), Foundation 8 cards, Applications 4 cards, footer
- [x] **No placeholders**: all code is complete; no TBD/TODO/similar
- [x] **quarkmind URL**: correctly uses `github.com/mdproctor/quarkmind` (not casehubio)
- [x] **CSS paths**: `/assets/css/main.css` — absolute path works when served from root; Python server in Task 3 Step 1 verifies this
- [x] **Jekyll migration**: `assets/css/main.css` path and `index.html` at root match Jekyll defaults exactly
- [x] **No cd before git**: all git commands use `git -C <path>`
- [x] **Apps-bg CSS**: `.apps-bg` wraps the application cards section with `bg-card` background; inner `.project-card` uses `bg-deep` for differentiation (reversed vs Foundation section)
