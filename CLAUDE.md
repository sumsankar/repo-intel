# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

repo-intel is a prompt-driven AI repository analysis platform. It ingests software repositories (GitHub, GitLab, Azure DevOps) and generates structured intelligence reports across six dimensions: code quality, architecture, security, DevOps, dependency risk, and governance. There is no traditional source code — the system is composed entirely of Markdown skill definitions and prompt orchestration, executed via Claude Code CLI.

## How It Works

The entry point is **ANALYZE.md**, which orchestrates the full analysis pipeline:
1. Accepts a remote repo URL or local folder path
2. Loads skill modules from `skills/*.md`
3. Executes skills sequentially: Security → Code → Architecture → DevOps → Dependency → Governance → Claude Metrics
4. Generates a self-contained HTML report (`repo-intel-report.html`) using `output/report-template.html`
5. Cleans up cloned repos (remote URLs only)

Skills are pluggable — each `skills/*.md` file defines an independent analysis dimension. See `skills/HOW-TO-ADD-SKILL.md` for adding new ones.

## CI and Linting

CI runs on GitHub Actions (`.github/workflows/ci.yml`) on every push to main and all PRs.

```bash
# Install linting tools
npm install -g markdownlint-cli2 markdown-link-check

# Lint all Markdown
markdownlint-cli2 "**/*.md" "#node_modules"

# Check links in a specific file
markdown-link-check <file.md>
```

## Key Directories

- `skills/` — Analysis skill modules (the core logic of the platform)
- `output/` — Report templates (HTML and Markdown)
- `docs/` — Platform design documentation (data pipeline, knowledge graph, API, scaling)
- `architecture/` — System architecture design (7-layer model, microservices, data flow)
- `engineering/` — Coding standards and module design guidelines
- `agents/` — AI agent specifications for each analysis domain
- `examples/` — Sample generated reports

## Architecture Principles

- AI augments static analysis — AI reasons over structured findings, not raw code
- Skill-based extensibility — new analysis dimensions don't require changes to the orchestration core
- Skills run independently — one failure doesn't cascade to others
- Ephemeral clones — no persistent storage of analyzed repository code
