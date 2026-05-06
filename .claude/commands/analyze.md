---
description: Run a repo-intel analysis on a remote repo URL or local folder path
argument-hint: <repo-url-or-path>
---

Run a full repo-intel analysis on: $ARGUMENTS

If `$ARGUMENTS` is empty, ask the user for a repo URL or local folder path before doing anything else.

1. Read `ANALYZE.md` at the project root and follow it end-to-end — it is the canonical orchestration contract. Do not skip steps.
2. For **Phase 2 parallel analysis**, prefer these project-scoped subagents over generic `Agent` calls. Spawn all four in a **single message** so they run concurrently:
   - `ri-code` — size, coverage, complexity, duplication, documentation
   - `ri-architecture` — structure, dependencies, patterns, diagrams
   - `ri-devops` — CI/CD, containers, IaC, hygiene
   - `ri-dependency` — CVEs, licences, supply chain, deprecated packages
3. Phase 1 (`security`) and Phase 3 (`governance`, `claude-metrics`) run in the main agent per `ANALYZE.md`.
4. Emit SARIF first, then render HTML (and Markdown if configured). The SARIF is canonical — validate it against `output/report.schema.json` before rendering.
