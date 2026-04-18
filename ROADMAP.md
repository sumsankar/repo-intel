# Roadmap

This file tracks capabilities that have been described in design documents but are **not implemented**. Today repo-intel is a local Claude Code prompt bundle (see [ARCHITECTURE.md](ARCHITECTURE.md)). The items below are speculative and unscheduled unless explicitly marked otherwise.

---

## Near-term (next 1–2 releases)

### Fixture-based regression tests
- Add 3 small fixture repos under `examples/fixtures/` (polyglot, monorepo, .NET solution) with planted issues.
- CI analyses each and diffs the resulting SARIF against a committed golden file.
- Catches skill drift before it ships.

### Diff / PR mode
- `--since=<ref>` flag to analyse only files changed since a git ref.
- Makes repo-intel usable as a PR-annotation bot (paired with the existing SARIF → Code Scanning upload).

### Comparison mode
- [ANALYZE.md](ANALYZE.md) advertises "compare two repos" but has no implementation. Specify the diff semantics (score deltas, added/resolved findings) and add a `compare` subcommand path in the orchestrator.

### Additional skills
From [skills/HOW-TO-ADD-SKILL.md](skills/HOW-TO-ADD-SKILL.md):

| Skill | Priority | What it would check |
|-------|----------|---------------------|
| `performance.md`   | High   | N+1 queries, bundle size, memory leaks, async anti-patterns |
| `api-design.md`    | Medium | REST conventions, versioning, OpenAPI validation, rate limiting |
| `database.md`      | Medium | Migrations, ORM usage, index coverage, raw SQL patterns |
| `logging.md`       | Medium | Structured logging, sensitive data in logs, correlation IDs |
| `accessibility.md` | Medium | Missing alt tags, ARIA roles, contrast, keyboard nav |
| `testing.md`       | Medium | Test quality, mocking, integration vs unit ratio |
| `documentation.md` | Low    | JSDoc/docstring coverage, stale comments, changelog accuracy |
| `i18n.md`          | Low    | Hardcoded strings, locale files, RTL support |

---

## Medium-term (Phase 2 — service-based platform)

If repo-intel grows beyond a local CLI tool, the following components become relevant. None of them exist today; references to them in older design documents reflect aspirational planning, not current architecture.

### HTTP API (FastAPI or ASP.NET Core)
- `POST /analyses` — submit a repo URL, get an analysis ID
- `GET  /analyses/{id}` — poll status
- `GET  /analyses/{id}/report.sarif` — download canonical output
- `POST /webhooks/github` — auto-analyse on push / PR
- Authentication, rate limiting, tenant isolation

### Worker pool
- Queue-backed (Redis / SQS / Azure Service Bus) so long analyses don't block the API.
- Horizontal scaling per queue depth.
- Timeout + cancellation semantics.

### Persistent storage
- Metadata store (PostgreSQL): analyses, users, tenants, quotas.
- Report store (blob / S3): SARIF, HTML, MD outputs, retained per policy.
- **Optional** knowledge graph (Neo4j or a property graph on PostgreSQL) for cross-analysis queries: "show me every repo in the org with a `RI-SEC-001` finding this quarter."

### Observability
- Structured logging with correlation IDs per analysis.
- Metrics: p50/p95/p99 analysis duration, token cost per run, finding counts per rule.
- SLOs: 95% of analyses < 5 minutes; 99.5% successful.
- Error budgets tied to rule-level precision/recall on fixture corpus.

### Cost model
- Token cost estimate per run, visible to users before they submit.
- Per-tenant quotas (tokens/month, analyses/day).
- Caching: if `commit SHA + config hash` matches a prior run, return the cached report.

### Deployment
- Containerised services.
- Kubernetes manifests or Helm chart.
- CI/CD pipeline for the platform itself (not to be confused with CI that uses repo-intel).

### Governance engine
- Persistent policy store versioned like code.
- Policy changes are PRs with CODEOWNERS approval.
- Compliance drift tracking across repos over time.

---

## What will NOT be built

To keep scope honest, the following are explicitly out of scope:

- **A web dashboard.** SARIF output plus the existing HTML report is sufficient. GitHub Code Scanning, Azure DevOps, and SonarQube already render SARIF well.
- **Arbitrary code execution of the target repo.** repo-intel is a static analyser. We do not build, run, or install dependencies from analysed repos.
- **AI-authored fixes.** We describe fixes; humans (or their editors) apply them.

---

## Decision log

Architectural decisions and their rationale are tracked here going forward. New entries go at the top.

| Date | Decision | Rationale |
|------|----------|-----------|
| 2026-04-18 | SARIF 2.1.0 as canonical output format | Plugs into GitHub Code Scanning, Azure DevOps, Defender for DevOps, and most SAST dashboards without custom integration. |
| 2026-04-18 | Dimension weights rebalanced to include `governance` (0.05) and split `code` 0.25 → 0.20 | Previous formula summed to 1.00 across only 5 of 7 skills; governance was excluded from the aggregate despite being a scoring skill. |
| 2026-04-18 | `RI-*` rule-ID namespace adopted | Stable IDs enable per-rule overrides in `repo-intel.yml` and fingerprint-based dedup across runs. |
