# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

repo-intel is a **local Claude Code skill bundle** — not a service. Pointed at any software repository (remote URL or local path), it produces a structured intelligence report (SARIF 2.1.0 canonical + HTML/Markdown renderings) across six scoring dimensions: security, code quality, architecture, DevOps, dependency risk, and governance.

The entire system is Markdown prompts plus one HTML template. There is no application code, database, API, or worker pool — see [ARCHITECTURE.md](ARCHITECTURE.md) for what exists today and [ROADMAP.md](ROADMAP.md) for Phase-2 ideas.

## How It Works

The entry point is [ANALYZE.md](ANALYZE.md), which orchestrates the analysis pipeline:

1. Locate the target repo (clone if a URL, else use the local path)
1.5. Load `repo-intel.yml` (validated against [output/config.schema.json](output/config.schema.json)) and merge `.repo-intelignore`
1.6. Discovery scan — tech stack, projects, monorepo layout
2. Load contract files: [SCORING-CONTRACT.md](skills/SCORING-CONTRACT.md), [FINDING-SCHEMA.md](skills/FINDING-SCHEMA.md), [SUBAGENT-OUTPUT.md](skills/SUBAGENT-OUTPUT.md)
3. Run skills in phases:
   - **Phase 1** (main agent): `security` — blocking, escalates critical secrets immediately
   - **Phase 2** (parallel subagents): `code`, `architecture`, `devops`, `dependency`
   - **Phase 3** (sequential): `governance`, `claude-metrics` (meta-skill, no score)
4. Cross-skill synthesis — root causes, patterns, quick wins
5. Build SARIF → validate against [output/report.schema.json](output/report.schema.json) → render HTML → (optional) Markdown
6. Cleanup cloned repos

Skills are pluggable. Adding one is a Markdown edit plus rule registration in [FINDING-SCHEMA.md](skills/FINDING-SCHEMA.md). See [skills/HOW-TO-ADD-SKILL.md](skills/HOW-TO-ADD-SKILL.md).

## Key contracts

Every skill must conform to these. A change to any of them is a semver-significant event.

| File | Role |
|------|------|
| [skills/SCORING-CONTRACT.md](skills/SCORING-CONTRACT.md) | Scoring model. Dimension weights sum to 1.00 across the six scoring skills. |
| [skills/FINDING-SCHEMA.md](skills/FINDING-SCHEMA.md) | Canonical `RI-*` rule registry (e.g. `RI-SEC-001-HARDCODED-SECRET`). |
| [skills/SUBAGENT-OUTPUT.md](skills/SUBAGENT-OUTPUT.md) | YAML wire format skills return to the main agent. |
| [output/config.schema.json](output/config.schema.json) | JSON Schema for `repo-intel.yml`. |
| [output/report.schema.json](output/report.schema.json) | SARIF 2.1.0 + `repo-intel.*` extension properties. |

## Analysis scope

Analyses **every project in a solution** — all `.csproj` files, all Maven modules, all monorepo packages. Never truncates findings to "top N". Path exclusions come from `repo-intel.yml` and `.repo-intelignore` only.

Language/framework coverage: .NET/C#, Java/Kotlin (Spring Boot), Python (Django, Flask), Node.js/TypeScript (React, Next.js, Angular), Go, Rust, Ruby/Rails, PHP/Laravel, Dart/Flutter, and more.

## CI and linting

CI runs on GitHub Actions ([.github/workflows/ci.yml](.github/workflows/ci.yml)) on every push to main and all PRs.

```bash
# Install linting tools
npm install -g markdownlint-cli2 markdown-link-check

# Lint all Markdown
markdownlint-cli2 "**/*.md" "#node_modules"

# Check links in a specific file
markdown-link-check <file.md>
```

## Key directories

- [skills/](skills/) — analysis skill modules and their shared contracts (the core of the system)
- [output/](output/) — config schema, SARIF schema, HTML + Markdown report templates, example config
- [examples/](examples/) — sample generated reports
- [.github/](.github/) — CI workflow, Dependabot, SECURITY.md
- [.claude/](.claude/) — Claude Code native wrappers: `commands/analyze.md` slash command and `agents/ri-{code,architecture,devops,dependency}.md` Phase 2 subagents. Thin wrappers over `ANALYZE.md` and `skills/*.md` — those remain the contract surface.

Nothing else exists at the root beyond entry-point docs (`ANALYZE.md`, `ARCHITECTURE.md`, `ROADMAP.md`, `README.md`, `CHANGELOG.md`, `CONTRIBUTING.md`, `LICENSE`).

## Architecture principles

- **SARIF is canonical.** HTML and Markdown are derived renderings. All dashboards consume the SARIF.
- **Contract-first skills.** Skills share one scoring model, one rule registry, one wire format. No per-skill dialects.
- **Skill-based extensibility.** New dimensions land as Markdown edits; the orchestrator does not change.
- **Parallel execution.** Independent skills run as Claude subagents for throughput.
- **Failure isolation.** A skipped or failing skill is renormalised out of the aggregate — never blocks others.
- **Exhaustive analysis.** All projects, all files, all findings; never cap output.
- **Ephemeral clones.** Remote repos are cloned `--depth 1`, analysed, then deleted. No persistent storage of analysed repo content.
- **No runtime execution.** Static analysis only — we do not build, run, or install dependencies from the analysed repo.
