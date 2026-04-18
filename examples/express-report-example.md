# Repository Intelligence Report

**Repository:** https://github.com/expressjs/express
**Analysed:** 2026-04-18 10:30 UTC
**repo-intel version:** 1.0.0
**Skills run:** security, code, architecture, devops, dependency, governance
**Canonical artefact:** [reports/repo-intel.sarif](../reports/repo-intel.sarif)

> This file is the Markdown rendering. The SARIF 2.1.0 document is the source of truth — upload it to GitHub Code Scanning, Azure DevOps, or any SARIF consumer. HTML version: `reports/repo-intel.html`.

---

## Executive summary

Express.js is a mature, widely-used Node.js web framework (~15 k LOC, 85 files). The project is healthy overall — strong test ratio, lean dependency tree, clear layering — but CI is missing a security-scan step and three workflow files pin outdated `actions/checkout@v2`. Governance flags one policy failure: no SAST/SCA step violates `RI-GOV-003-POLICY-FAILURE-CI`. Highest-priority fix is wiring a minimal vulnerability scan into `.github/workflows/ci.yml`.

---

## Health scorecard

Weights come from [`SCORING-CONTRACT.md`](../skills/SCORING-CONTRACT.md) §1.

| Dimension    | Score | Weight | Weighted | Status       |
|--------------|------:|-------:|---------:|--------------|
| Security     |   6.8 |   0.30 |     2.04 | Needs work   |
| Code quality |   8.2 |   0.20 |     1.64 | Good         |
| Architecture |   8.0 |   0.20 |     1.60 | Good         |
| DevOps       |   5.4 |   0.15 |     0.81 | Needs work   |
| Dependency   |   8.5 |   0.10 |     0.85 | Good         |
| Governance   |   6.5 |   0.05 |     0.33 | Needs work   |
| **Overall**  | **7.27** | 1.00 | **7.27** | **Needs work** |

Score bands: 8.0–10 good · 5.0–7.9 needs work · 3.0–4.9 poor · 0.0–2.9 critical.

---

## Repository overview

| Metric | Value |
|--------|-------|
| Primary language | JavaScript |
| Project type | Node.js library |
| Total files | 85 |
| Total LOC | 14,820 |
| Test files | 31 |
| Test ratio | 36.5% |
| CI/CD | GitHub Actions |
| Container | None (library) |
| IaC | None |
| Direct deps | 8 |
| Lock file | `package-lock.json` present |

---

## Findings summary

| Severity | Count |
|----------|------:|
| Critical |     0 |
| High     |     3 |
| Medium   |     3 |
| Low      |     4 |
| **Total**|    **10** |

---

## High-severity findings

### RI-DEVOPS-003-NO-SECURITY-SCAN — No SAST / secret scan / SCA step in CI
- **File:** `.github/workflows/ci.yml`
- **Severity:** high · CWE-1357
- **Message:** CI runs tests but never audits dependencies or scans for secrets, so vulnerable packages and accidental credential commits are never caught automatically.
- **Fix:** Add `- run: npm audit --audit-level=high` and a Gitleaks action step to `.github/workflows/ci.yml`.

### RI-DEVOPS-002-OUTDATED-ACTIONS — `actions/*@v1` or `@v2` references
- **Files:** `.github/workflows/ci.yml:14`, `.github/workflows/test.yml:11`, `.github/workflows/lint.yml:10`
- **Severity:** high
- **Message:** Three workflow files reference `actions/checkout@v2`; v2 is no longer receiving security backports.
- **Fix:** Bump all occurrences to `actions/checkout@v4`.

### RI-GOV-003-POLICY-FAILURE-CI — No SAST/SCA step in CI
- **Severity:** high
- **Message:** Governance policy requires every repo in the `libraries` tier to run at least one SAST or SCA tool in CI. This repo does not, wrapping `RI-DEVOPS-003`.
- **Fix:** Resolving `RI-DEVOPS-003` clears this automatically.

---

## Medium-severity findings

### RI-CODE-001-LARGE-FILE
- **File:** `lib/router/index.js` (612 lines)
- **Fix:** Extract route-matching logic into `lib/router/match.js`; keep registration in `index.js`.

### RI-SEC-011-SWALLOWED-EXCEPTION *(actually low — promoted here by `thresholds.upgrade` in repo-intel.yml)*
- **File:** `lib/utils.js:89`
- **Evidence:** `try { return JSON.parse(...); } catch (e) {}`
- **Fix:** Log the parse failure; rethrow if unexpected.

### RI-CODE-008-HIGH-TODO-DENSITY
- **Files:** `lib/application.js`, `lib/response.js`, `lib/utils.js` (4 TODOs total)
- **Fix:** Link each TODO to a tracked GitHub issue or resolve.

---

## Low-severity findings

- **RI-SEC-009-MISSING-GITIGNORE-SECRETS** — `.env` pattern not in `.gitignore`. Add it even though no `.env` file exists today.
- **RI-CODE-007-MISSING-README** variant — `examples/` directory has no index/README. Add a one-paragraph guide.
- **RI-DEP-007-OUTDATED-MAJOR** — `debug@2.x` is 2 majors behind `4.x`.
- **RI-GOV-005-POLICY-FAILURE-DOCS** — No `SECURITY.md` for vulnerability disclosure policy.

---

## Per-dimension detail

### Security · 6.8 / 10
No hardcoded credentials, no SQL/command-injection patterns, no weak crypto detected. Deductions come from RI-SEC-009 (gitignore) and the swallowed-exception upgrade noted above.

### Code quality · 8.2 / 10
31 test files against 82 source files = 36.5% ratio (healthy for a library). One large file and a handful of TODOs account for the deductions.

### Architecture · 8.0 / 10
Clear `lib/` + `test/` layering, 8 direct dependencies, no circular imports. No deductions from layering rules; minor deduction from RI-ARCH-005 (inconsistent error-handling patterns across `lib/response.js` and `lib/request.js`).

### DevOps · 5.4 / 10
GitHub Actions present, tests run, lock file present. Two high findings (`RI-DEVOPS-002`, `RI-DEVOPS-003`) cap this dimension.

### Dependency · 8.5 / 10
No known CVEs in the current `package-lock.json`. One `RI-DEP-007` (debug@2).

### Governance · 6.5 / 10
Policy failures: `RI-GOV-003` (CI), `RI-GOV-005` (docs). Each wraps an underlying finding in another dimension.

---

## Quick wins (top 3)

1. **Add `npm audit` + Gitleaks to CI** — resolves `RI-DEVOPS-003` and `RI-GOV-003`. One-line each.
2. **Bump `actions/checkout` to v4** — resolves `RI-DEVOPS-002` across three files.
3. **Add `.github/SECURITY.md`** — resolves `RI-GOV-005`. GitHub surfaces it automatically.

---

## Recommended roadmap

### This week
- Resolve `RI-DEVOPS-002` and `RI-DEVOPS-003` (also clears `RI-GOV-003`).
- Add `.github/SECURITY.md` (clears `RI-GOV-005`).

### This month
- Split `lib/router/index.js` (resolves `RI-CODE-001`).
- Replace swallowed exception in `lib/utils.js:89` (resolves `RI-SEC-011`).
- Configure Dependabot and enable `RI-DEP-*` automated PRs.

### This quarter
- Add CodeQL for deeper `RI-SEC-*` coverage.
- Bump `debug` to v4 (resolves `RI-DEP-007`).

---

## Exit status

`thresholds.fail_on: high` is set in `repo-intel.yml`. This run emitted 3 high-severity findings → exit code **1**.

---

*Rendered from `reports/repo-intel.sarif` · repo-intel 1.0.0*
