# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

Contract files — [`skills/SCORING-CONTRACT.md`](skills/SCORING-CONTRACT.md), [`skills/FINDING-SCHEMA.md`](skills/FINDING-SCHEMA.md), [`skills/SUBAGENT-OUTPUT.md`](skills/SUBAGENT-OUTPUT.md), [`output/config.schema.json`](output/config.schema.json), [`output/report.schema.json`](output/report.schema.json) — are semver-significant: a breaking change to any of them is a major bump.

## [1.0.0] — 2026-04-18

First tagged release. Establishes stable contracts, SARIF-canonical output, and a reality-aligned doc set.

### Added

**Canonical contracts**
- [`skills/SCORING-CONTRACT.md`](skills/SCORING-CONTRACT.md) — single scoring model across all skills. Severity multipliers, per-rule cap, dimension weights summing to 1.00, renormalisation formula for skipped skills.
- [`skills/FINDING-SCHEMA.md`](skills/FINDING-SCHEMA.md) — canonical `RI-*` rule registry. 49 rules across 6 skills, each with default severity, CWE/OWASP mapping, and base impact.
- [`skills/SUBAGENT-OUTPUT.md`](skills/SUBAGENT-OUTPUT.md) — YAML wire format skills return to the orchestrator. Size caps, HTML-escaping requirement, partial-failure semantics.

**Config + output schemas**
- [`output/config.schema.json`](output/config.schema.json) — JSON Schema (Draft 2020-12) for `repo-intel.yml`. Covers skills selection, exclude/include globs, thresholds, per-rule overrides, secrets allowlist, output formats, token budget.
- [`output/report.schema.json`](output/report.schema.json) — SARIF 2.1.0 with `repo-intel.*` extension properties: schemaVersion, scores, repoContext, findingsSummary, skippedSkills. Enforces `RI-*` rule-ID pattern.
- [`output/repo-intel.example.yml`](output/repo-intel.example.yml) — annotated reference config with `$schema` header.
- [`output/.repo-intelignore.example`](output/.repo-intelignore.example) — gitignore-style path exclusion template.

**Architecture docs**
- [`ARCHITECTURE.md`](ARCHITECTURE.md) — describes the local-CLI system that actually exists today. Component diagram, execution pipeline, contracts list.
- [`ROADMAP.md`](ROADMAP.md) — Phase-2 service-based platform, additional skills backlog, decision log.

**CI**
- JSON Schema validation job — compiles `config.schema.json` and `report.schema.json`, validates `repo-intel.example.yml` against the config schema.
- Gitleaks secret-scan job.
- HTML template sanity job — checks Mermaid CDN pin + SRI presence.
- Artefact guard job — blocks checked-in `*repo-intel-report*` files.

### Changed

- **[`ANALYZE.md`](ANALYZE.md)** — added Step 1.5 (load `repo-intel.yml`, merge `.repo-intelignore`) and Step 1.6 (discovery scan). Step 5 rewritten as SARIF-first: build SARIF → validate against schema → render HTML from SARIF → optional Markdown. Exit codes formalised (0 clean, 1 threshold breached, 2 analyser error).
- **All 7 skill files** — each now declares which `RI-*` rules it emits, pointing to `FINDING-SCHEMA.md` and `SUBAGENT-OUTPUT.md`. Skill-specific scoring conventions removed in favour of the central contract.
- **Dimension weights rebalanced** — `security` 0.30, `code` 0.20, `architecture` 0.20, `devops` 0.15, `dependency` 0.10, `governance` 0.05. Previous formula summed to 1.00 across only 5 of 7 skills; `governance` was excluded from the aggregate despite being a scoring skill. `claude-metrics` is now formally classified as a meta-skill with no score.
- **[`output/report-template.html`](output/report-template.html)** — Mermaid pinned to `mermaid@11.4.1` with `integrity`, `crossorigin`, `referrerpolicy`, and an `onerror` fallback. Modal SVG open path replaced `innerHTML` assignment with a `cloneNode(true)` DOM copy to close a DOM-XSS sink on user-controlled Mermaid text.
- **`.gitignore`** — tightened patterns for generated reports, added `/reports/`, added `* - Copy.*` / `*.copy.*` patterns.
- **[`README.md`](README.md)** — rewritten as a local-CLI quick-start with honest scope. Dimension/weight table, repo layout, SARIF-first output explanation, exit-code reference, explicit "what it is not" section.
- **[`CLAUDE.md`](CLAUDE.md)** — rewritten to match the current layout and contracts.
- **[`examples/express-report-example.md`](examples/express-report-example.md)** — updated to the 6-dimension scorecard with `RI-*` rule IDs and a note that the SARIF document is canonical.

### Removed

- `agents/` (4 files) — duplicated `skills/` logic.
- `docs/` (19 files) — overwhelmingly Phase-2 aspirational. Load-bearing content merged into `ARCHITECTURE.md` / `ROADMAP.md`.
- `architecture/` (4 files) — overlapping architecture drafts; consolidated into `ARCHITECTURE.md`.
- `engineering/` (3 files) — coding standards for application code that does not exist in this repo.
- `hrms-angular_repo-intel-report.html`, `repo-intel-report - Copy.html`, `output/report-repo-intel-md.html` — leaked generated artefacts.
- `infographic.svg` — orphan marketing asset.

### Fixed

- Dimension weights no longer sum to ≠1.00 when all skills run.
- Renormalisation on skipped skills now removes the weight instead of scoring the skill as 0.
- DOM-XSS risk in the HTML report modal when Mermaid-rendered SVGs contain user-controlled repo text.
- Supply-chain exposure on the Mermaid CDN reference (floating tag, no SRI).

## [0.x] — pre-1.0

Undocumented. Early skill drafts, report template, and initial `ANALYZE.md`. See git history for details.
