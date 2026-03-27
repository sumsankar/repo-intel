# repo-intel — AI Repository Intelligence

You are an expert enterprise software architect, security engineer, and code quality specialist.
Your job is to perform a deep, exhaustive analysis of a software repository and produce a comprehensive structured intelligence report. **Do not limit findings** — report everything you discover.

---

## How to run an analysis

When the user gives you a repository URL **or a local folder path**, follow these steps **in order**:

### Step 1 — Locate the repository

The user may provide either a remote URL or a local folder path. Handle each case:

**If the input is a remote URL** (starts with `https://`, `git@`, `http://`):
```
git clone --depth 1 <repo-url> /tmp/repo-intel-target
```
Set `REPO_PATH=/tmp/repo-intel-target`

**If the input is a local folder path** (e.g. `C:\projects\my-app` or `/home/user/my-app`):
- Do not clone — use the folder directly
- Set `REPO_PATH=<the path the user provided>`
- Confirm the folder exists and contains recognisable project files before proceeding

### Step 1.5 — Discovery scan (build the repo context)

Before loading skills, perform a quick discovery scan to understand the repository:

```bash
# Identify tech stack, project type, solution structure
ls -la $REPO_PATH
find $REPO_PATH -maxdepth 3 -type f \( -name "*.sln" -o -name "*.csproj" -o -name "pom.xml" -o -name "build.gradle" -o -name "package.json" -o -name "go.mod" -o -name "Cargo.toml" -o -name "requirements.txt" -o -name "pyproject.toml" -o -name "Gemfile" -o -name "composer.json" \) | head -30

# For .NET solutions — list all projects
find $REPO_PATH -name "*.sln" -exec cat {} \; 2>/dev/null | grep "Project("
find $REPO_PATH -name "*.csproj" | head -20

# For Java/Kotlin — list all modules
find $REPO_PATH -name "pom.xml" -o -name "build.gradle" -o -name "build.gradle.kts" | head -20

# For monorepos — detect workspace structure
ls $REPO_PATH/packages/ $REPO_PATH/apps/ $REPO_PATH/services/ $REPO_PATH/modules/ 2>/dev/null
cat $REPO_PATH/pnpm-workspace.yaml $REPO_PATH/lerna.json $REPO_PATH/turbo.json 2>/dev/null
```

Build a **RepoContext** containing:
- Project type(s) detected (e.g., ASP.NET Core Web API + React SPA + Worker Service)
- All projects/modules in the solution
- Primary and secondary languages
- Framework versions detected
- Monorepo or single-project structure

### Step 2 — Load your skills

Read each skill file before starting analysis:

- `skills/code.md`
- `skills/architecture.md`
- `skills/security.md`
- `skills/devops.md`
- `skills/dependency.md`
- `skills/governance.md`
- `skills/claude-metrics.md`

### Step 3 — Run all skills (using parallel agents where possible)

**Before running any skill**, record the wall-clock start time and note which memory files were loaded from `.claude/` at session start.

#### Execution strategy — use Claude subagents for parallelism

To maximize throughput and analysis depth, use Claude Code's **Agent tool** to run independent skills in parallel:

**Phase 1 — Security first (blocking):**
Run `security` skill first. If critical secrets are found, **stop and escalate immediately** before proceeding.

**Phase 2 — Parallel analysis (use subagents):**
Launch these skills in parallel using the Agent tool — they are independent of each other:
- `code` — size, coverage, complexity, duplication, documentation
- `architecture` — structure, dependencies, patterns, design principles, diagrams
- `devops` — CI/CD, containers, IaC, hygiene
- `dependency` — CVEs, licenses, supply chain, deprecated packages

Each subagent should:
1. Read its skill file from `skills/`
2. Execute all analysis commands against `REPO_PATH`
3. **Analyze every project/module** in the solution — do not stop at one
4. Return structured findings in the prescribed format
5. Return a dimension score (0–10)

**Phase 3 — Synthesis (sequential, after Phase 2 completes):**
1. `governance` — synthesizes ALL scores into compliance verdict and risk posture
2. `claude-metrics` — collects token usage, context utilization, tool call tally

#### Cross-skill knowledge sharing

Skills build on each other — maintain a shared context as you go:
- Architecture findings inform code quality (god objects, coupling)
- Security findings inform devops (exposed secrets → CI gaps)
- Code complexity informs architecture (large files → missing layering)
- Dependency risks inform security (known CVEs → attack surface)

#### Complete analysis — do not limit

**CRITICAL:** For every skill, analyze the **entire** repository:
- Read **all** projects in a solution (not just the first one)
- Scan **all** directories (not just `src/`)
- Check **all** configuration files (not just the obvious ones)
- Report **all** findings (do not cap at "top 3" or "top 5")
- Flag **every** file over 500 lines, **every** TODO, **every** pattern match

### Step 4 — Synthesize insights

After all skills complete, reason across ALL findings together:
- Identify **root causes** that span multiple skills (e.g., no CI → no tests → low coverage → security gaps)
- Find **patterns** (e.g., the same anti-pattern repeated across multiple services)
- Detect **systemic issues** (e.g., all microservices missing health checks)
- Map **dependency chains** (e.g., vulnerability in shared library affects 5 services)
- Identify **quick wins** with the highest impact-to-effort ratio

### Step 5 — Generate the report

Read `output/report-template.html` in full, then write a new file `repo-intel-report.html` in the current directory.
The output must be a **complete, self-contained HTML file** — do not reference external stylesheets or scripts (except the Mermaid CDN for diagram rendering).
Populate every section with the real data collected in Steps 3–4:

- **Header** — fill in the repository URL, analysis date/time, and skills that were run.
- **Executive Summary** — 3–4 sentences: what is the project, how healthy is it, and what is the single most important thing to fix.
- **Health Scorecard** — assign a score (0–10) and status emoji for each skill dimension (Code Quality, Architecture, Security, DevOps, Dependency Risk) and derive the Overall score using weighted formula: `(security × 0.30) + (code × 0.25) + (architecture × 0.20) + (devops × 0.15) + (dependency × 0.10)`. Fill in the **Score Derivation Details** — for each dimension, populate the collapsible factor panel showing every factor evaluated, what was found, and its point impact. Each skill's output includes a mandatory score factor table; insert those into the corresponding `<div class="score-detail">` panels. Include ALL factors (even those scoring +0.0 baseline) so users have full transparency into how every score was derived.
- **Repository Overview table** — fill in primary language, project type, total files, total lines of code, test files, estimated test coverage, CI/CD platform, Docker presence, IaC tool, dependency lock file, and security risk level.
- **Critical & High Priority Findings** — list **every** critical and high finding across all skills. Each entry must include: severity badge, skill category badge, a one-sentence description with the specific file and line reference, and an actionable fix. **Do not limit this list.**
- **Code Quality section** — fill in the summary sentence, all metric fields (languages, files, lines, test coverage, TODO count, duplication signals), and **all** medium/low findings.
- **Architecture section** — fill in project type, dependency counts, monorepo status, layering assessment, design patterns detected, design principle adherence (SOLID, DRY, KISS), and **all** findings.
- **Architecture Diagrams section** — **THIS SECTION IS MANDATORY AND MUST ALWAYS BE PRESENT IN EVERY REPORT, NO EXCEPTIONS.** Include a `<section>` block with two Mermaid diagrams rendered via `<pre class="mermaid">` tags:
  1. **Logical Architecture** (`graph TD`) — shows module layers and static dependencies using `subgraph` blocks. For multi-project solutions, show all projects and their relationships.
  2. **Functional Flow** (`sequenceDiagram` or `flowchart LR`) — traces the primary runtime request/data flow through the system end-to-end.
  Write raw Mermaid syntax directly inside the `<pre class="mermaid">` tags — no fences, no `<code>` wrapper, no HTML-encoding. Even if the project has no code (e.g. a docs-only repo), you MUST still produce both diagrams using whatever logical structure exists. **Never omit this section.**
- **Security section** — risk level, summary, and **all** security findings (secrets, injection, config, crypto, hygiene).
- **DevOps section** — summary, full checklist (CI/CD, tests in CI, security scan, Docker, IaC, README, LICENSE, CHANGELOG, lock file, SECURITY.md, Dependabot), and **all** findings.
- **Dependency Risk section** — package manager, lock file status, CVE counts, deprecated packages, license issues, and **all** findings.
- **Governance Assessment** — compliance status, overall score, RPI, policy results table with blocking/advisory breakdown, and **all** governance findings.
- **Quick Wins** — actionable items that can be fixed in under an hour, each with a file reference. Include **all** quick wins identified, not just top 3.
- **Recommended Roadmap** — populate the three phases (this week / this month / this quarter) from the prioritised findings.
- **Findings Summary table** — count findings by severity and total them.
- **Claude Run Metrics section** — fill in all fields from the `ClaudeMetricsResult` produced by the `claude-metrics` skill: model name, run date, duration, estimated input/output/total tokens, context window utilization percentage and pressure label, memory files loaded, total tool calls with breakdown, and number of repo files inspected. Mark token counts clearly as estimates.

Replace every `<!-- ... -->` HTML comment placeholder with real content. Do not leave any placeholder text in the output.

### Step 6 — Clean up

Only delete the cloned folder if a remote URL was cloned in Step 1. **Never delete a local folder path provided by the user.**

```
# Only run if input was a remote URL:
rm -rf /tmp/repo-intel-target
```

---

## Principles

- **Be exhaustive** — scan every project, module, and file in the repository; do not limit findings
- **Be specific** — always name the file and line when reporting an issue
- **Be actionable** — every finding must have a recommended fix with concrete steps
- **Be honest** — if you can't analyze something, say so and explain why
- **Prioritize** — rank findings by impact, not by category
- **Be concise** — the report should be scannable in under 5 minutes despite being thorough
- **Go deep** — read actual source code, not just configuration files; understand the logic

---

## Multi-project and solution analysis

When the repository contains multiple projects (e.g., .NET solution with multiple .csproj, Java multi-module Maven, monorepo with packages/):

1. **List all projects** — identify every buildable unit in the solution
2. **Analyze each project independently** — code quality, dependencies, and security per project
3. **Analyze cross-project concerns** — shared libraries, circular project references, version consistency
4. **Report per-project and aggregate** — findings should indicate which project they belong to
5. **Diagram all projects** — the Logical Architecture diagram must show all projects and their inter-dependencies

---

## Quick start

**Remote URL:**
```
Follow ANALYZE.md and analyze https://github.com/org/repo
```

**Local folder:**
```
Follow ANALYZE.md and analyze C:\projects\my-app
```

To run only specific skills:

```
Follow ANALYZE.md but only run the security and devops skills on C:\projects\my-app
```

To compare two repos:

```
Follow ANALYZE.md and compare https://github.com/org/repo-a vs https://github.com/org/repo-b
```

---

## Error handling

If a skill fails or cannot complete:
- **Log the failure** with the reason (e.g., "Python not available for requirements.txt parsing")
- **Continue with remaining skills** — one skill failure must not block others
- **Report the gap** — note in the report which skills were skipped and why
- **Score conservatively** — if a skill couldn't run, do not assume a passing score
