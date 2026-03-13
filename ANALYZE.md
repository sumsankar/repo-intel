# repo-intel — AI Repository Intelligence

You are an expert software architect and security engineer.
Your job is to analyze a GitHub or GitLab repository and produce a structured intelligence report.

---

## How to run an analysis

When the user gives you a repository URL, follow these steps **in order**:

### Step 1 — Clone the repository

```
git clone --depth 1 <repo-url> /tmp/repo-intel-target
cd /tmp/repo-intel-target
```

### Step 2 — Load your skills

Read each skill file before starting analysis:

- `skills/code.md`
- `skills/architecture.md`
- `skills/security.md`
- `skills/devops.md`
- `skills/dependency.md`
- `skills/governance.md`
- `skills/claude-metrics.md`

### Step 3 — Run all skills

**Before running any skill**, record the wall-clock start time and note which memory files were loaded from `.claude/` at session start.

Execute each skill against the cloned repo.
Write findings into a shared mental knowledge graph as you go.
Skills may build on each other — architecture informs code, security informs devops.

Run skills in this order:

1. `security` — always first; if critical secrets found, escalate immediately
2. `code` — size, coverage, complexity
3. `architecture` — structure, dependencies, patterns
4. `devops` — CI/CD, containers, hygiene
5. `dependency` — CVEs, licenses, supply chain
6. `governance` — runs LAST; synthesizes all scores into compliance verdict
7. `claude-metrics` — runs AFTER governance; collects token usage, context utilization, tool call tally, and memory file list for the current run

### Step 4 — Synthesize insights

After all skills complete, reason across ALL findings together.
Identify patterns, connections, and root causes that span multiple skills.

### Step 5 — Generate the report

Read `output/report-template.html` in full, then write a new file `repo-intel-report.html` in the current directory.
The output must be a **complete, self-contained HTML file** — do not reference external stylesheets or scripts.
Populate every section with the real data collected in Steps 3–4:

- **Header** — fill in the repository URL, analysis date/time, and skills that were run.
- **Executive Summary** — 3–4 sentences: what is the project, how healthy is it, and what is the single most important thing to fix.
- **Health Scorecard** — assign a score (0–10) and status emoji for each skill dimension (Code Quality, Architecture, Security, DevOps) and derive the Overall score as the average.
- **Repository Overview table** — fill in primary language, project type, total files, total lines of code, test files, estimated test coverage, CI/CD platform, Docker presence, IaC tool, and security risk level.
- **Critical & High Priority Findings** — list every critical and high finding across all skills. Each entry must include: severity badge, skill category badge, a one-sentence description with the specific file and line reference, and an actionable fix.
- **Code Quality, Architecture, Security, DevOps sections** — fill in the summary sentence, all metric fields, and the medium/low findings for that skill in the same badge format.
- **DevOps Checklist** — mark each item `pass` (green ✓) or `fail` (red ✗) based on what was found.
- **Top 3 Quick Wins** — three specific, actionable items that can be fixed in under an hour, each with a file reference.
- **Recommended Roadmap** — populate the three phases (this week / this month / this quarter) from the prioritised findings.
- **Findings Summary table** — count findings by severity and total them.
- **Claude Run Metrics section** — fill in all fields from the `ClaudeMetricsResult` produced by the `claude-metrics` skill: model name, run date, duration, estimated input/output/total tokens, context window utilization percentage and pressure label, memory files loaded, total tool calls with breakdown, and number of repo files inspected. Mark token counts clearly as estimates.

Replace every `<!-- ... -->` HTML comment placeholder with real content. Do not leave any placeholder text in the output.

### Step 6 — Clean up

```
rm -rf /tmp/repo-intel-target
```

---

## Principles

- **Be specific** — always name the file and line when reporting an issue
- **Be actionable** — every finding must have a recommended fix
- **Be honest** — if you can't analyze something, say so
- **Prioritize** — rank findings by impact, not by category
- **Be concise** — the report should be scannable in under 5 minutes

---

## Quick start

To analyze a repo, tell Claude Code:

```
Follow ANALYZE.md and analyze https://github.com/org/repo
```

To run only specific skills:

```
Follow ANALYZE.md but only run the security and devops skills on https://github.com/org/repo
```

To compare two repos:

```
Follow ANALYZE.md and compare https://github.com/org/repo-a vs https://github.com/org/repo-b
```
