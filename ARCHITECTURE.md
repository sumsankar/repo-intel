# Architecture

repo-intel is a **prompt-driven analysis pipeline** — a bundle of Markdown skill files orchestrated by Claude Code against a target repository. It is not a service, has no API, and runs entirely in the user's local Claude Code session.

This document describes what actually exists today. For the future service-based architecture, see [ROADMAP.md](ROADMAP.md).

---

## Component map

```
┌─────────────────────┐       ┌──────────────────────────┐
│  Target repository  │──────▶│       Claude Code         │
│  (local or cloned)  │       │                          │
└─────────────────────┘       │  reads ANALYZE.md         │
         ▲                    │        ↓                 │
         │ clone --depth 1    │  loads repo-intel.yml     │
         │                    │        ↓                 │
         │                    │  loads skill contracts    │
         │                    │  (SCORING, FINDING,       │
         │                    │   SUBAGENT-OUTPUT)        │
         │                    │        ↓                 │
         │                    │  runs skills via subagents│
         │                    │        ↓                 │
         │                    │  merges → SARIF           │
         │                    │        ↓                 │
         │                    │  renders HTML + MD        │
         │                    └─────────────┬────────────┘
         │                                  │
         └── (cleanup if cloned)            ▼
                                ┌──────────────────────┐
                                │  report.sarif (canon)│
                                │  report.html (render)│
                                │  report.md   (render)│
                                └──────────────────────┘
```

Everything in this diagram is Markdown or a single HTML template. There is no long-running process, no database, no message queue, no HTTP server.

---

## Execution pipeline

Entry point: [ANALYZE.md](ANALYZE.md). Claude Code is instructed to follow it step by step.

| Step | Purpose | Artefacts |
|------|---------|-----------|
| 1    | Locate target repo (clone if remote) | `$REPO_PATH` |
| 1.5  | Load `repo-intel.yml`, validate against [output/config.schema.json](output/config.schema.json), merge `.repo-intelignore` | `RUN_CONFIG` |
| 1.6  | Discovery scan — detect stack, projects, monorepo layout | `RepoContext` |
| 2    | Load skill contracts (SCORING / FINDING / SUBAGENT-OUTPUT) + Phase-1/3 skills | in-memory |
| 3    | Phase 1 security (main agent); Phase 2 code / architecture / devops / dependency (parallel subagents); Phase 3 governance / claude-metrics (sequential) | YAML per skill |
| 4    | Cross-skill synthesis — root causes, patterns, quick wins | merged |
| 5    | Build SARIF → validate against [output/report.schema.json](output/report.schema.json) → render HTML from SARIF → render MD (optional) | `report.sarif`, `report.html`, `report.md` |
| 6    | Cleanup (remote clones only) | — |

The SARIF document is the canonical output; HTML and Markdown are derived renderings.

---

## Key contracts

These are the files that keep skills composable:

| File | Role |
|------|------|
| [skills/SCORING-CONTRACT.md](skills/SCORING-CONTRACT.md) | One scoring model for all skills; dimension weights sum to 1.00 across all six scoring skills. |
| [skills/FINDING-SCHEMA.md](skills/FINDING-SCHEMA.md) | Canonical `RI-*` rule registry. Every finding uses one of these IDs. |
| [skills/SUBAGENT-OUTPUT.md](skills/SUBAGENT-OUTPUT.md) | YAML wire format each skill returns to the main agent. |
| [output/config.schema.json](output/config.schema.json) | JSON Schema for `repo-intel.yml`. |
| [output/report.schema.json](output/report.schema.json) | SARIF 2.1.0 + `repo-intel.*` extension properties. |
| [output/repo-intel.example.yml](output/repo-intel.example.yml) | Annotated reference config. |

A change to any contract is a semver-significant event. See the CHANGELOG for policy.

---

## Skills

A skill is a Markdown file in [skills/](skills/) that:

1. Describes what to analyse (in prose, for Claude to read).
2. Declares which `RI-*` rule IDs it can emit.
3. Returns a YAML envelope conforming to [SUBAGENT-OUTPUT.md](skills/SUBAGENT-OUTPUT.md).

Adding a new skill is a Markdown edit — no code. See [skills/HOW-TO-ADD-SKILL.md](skills/HOW-TO-ADD-SKILL.md).

Current skills:

| Skill | Phase | Weight | Meta? |
|-------|:-----:|-------:|:-----:|
| `security`       | 1 | 0.30 | — |
| `code`           | 2 | 0.20 | — |
| `architecture`   | 2 | 0.20 | — |
| `devops`         | 2 | 0.15 | — |
| `dependency`     | 2 | 0.10 | — |
| `governance`     | 3 | 0.05 | — |
| `claude-metrics` | 3 | —    | meta (no score) |

---

## What is not part of the system

To avoid confusion: the following have been referenced in older drafts of the docs but **are not implemented** and are not part of the architecture. They are roadmap ideas tracked in [ROADMAP.md](ROADMAP.md):

- No HTTP API or REST endpoints
- No database (PostgreSQL, Neo4j, Redis)
- No message queue or worker pool
- No Kubernetes deployment, containers, or cloud infrastructure
- No multi-tenancy or authentication layer
- No persistent knowledge graph
- No cross-run caching

If you need any of these capabilities today, you will need to build them yourself or wait for Phase 2.
