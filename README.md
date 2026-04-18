# repo-intel

> A Claude Code skill bundle that analyses any software repository and produces a structured intelligence report — SARIF, HTML, and Markdown — covering code quality, architecture, security, DevOps, dependency risk, and governance.

repo-intel is a **local CLI tool**, not a service. You point Claude Code at `ANALYZE.md`, give it a repo URL or folder path, and it emits a SARIF 2.1.0 document plus human-readable renderings. The SARIF can be uploaded to GitHub Code Scanning, Azure DevOps, Defender for DevOps, or any SARIF-compatible dashboard.

---

## Quick start

### Analyse a remote repo

```
Follow ANALYZE.md and analyse https://github.com/org/repo
```

### Analyse a local folder

```
Follow ANALYZE.md and analyse C:\projects\my-app
```

### Configure per-repo behaviour

Drop a [repo-intel.yml](output/repo-intel.example.yml) at the root of the target repo to control which skills run, path exclusions, CI thresholds, and rule-level overrides. See [output/config.schema.json](output/config.schema.json) for the full shape.

```yaml
# repo-intel.yml
version: 1
skills: [security, code, architecture, devops, dependency, governance]
exclude: ["**/node_modules/**", "**/dist/**"]
thresholds:
  fail_on: high
  min_scores: { overall: 6.0, security: 7.0 }
output:
  formats: [html, sarif]
```

### Gate a PR via GitHub Code Scanning

```yaml
# .github/workflows/repo-intel.yml (in the repo being analysed)
- name: Upload SARIF
  uses: github/codeql-action/upload-sarif@v3
  with:
    sarif_file: reports/repo-intel.sarif
```

---

## What it analyses

| Dimension | Weight | Rule namespace | What it looks at |
|-----------|:------:|----------------|------------------|
| Security      | 0.30 | `RI-SEC-*`    | Secrets, injection, auth, crypto, CORS, TLS |
| Code quality  | 0.20 | `RI-CODE-*`   | Size, tests, complexity, duplication, docs |
| Architecture  | 0.20 | `RI-ARCH-*`   | Layering, cycles, coupling, monorepo drift |
| DevOps        | 0.15 | `RI-DEVOPS-*` | CI/CD, Docker, IaC, lock files, security scans |
| Dependency    | 0.10 | `RI-DEP-*`    | CVEs, licences, deprecated packages, version skew |
| Governance    | 0.05 | `RI-GOV-*`    | Policy compliance wrapping the above |

Dimension weights sum to 1.00 and are defined once in [skills/SCORING-CONTRACT.md](skills/SCORING-CONTRACT.md). `claude-metrics` is a meta-skill — it records telemetry and contributes no score.

Full rule registry: [skills/FINDING-SCHEMA.md](skills/FINDING-SCHEMA.md).

---

## Repository layout

```
repo-intel/
├── README.md                       # This file
├── ANALYZE.md                      # Orchestration entry point — Claude Code follows this
├── ARCHITECTURE.md                 # What actually exists today
├── ROADMAP.md                      # What's planned, what won't be built
├── CHANGELOG.md
├── CONTRIBUTING.md
├── LICENSE
├── CLAUDE.md                       # Project hints for Claude Code
│
├── skills/
│   ├── SCORING-CONTRACT.md         # Single scoring model for all skills
│   ├── FINDING-SCHEMA.md           # Canonical RI-* rule registry
│   ├── SUBAGENT-OUTPUT.md          # YAML wire format skills return
│   ├── HOW-TO-ADD-SKILL.md
│   ├── security.md
│   ├── code.md
│   ├── architecture.md
│   ├── devops.md
│   ├── dependency.md
│   ├── governance.md
│   └── claude-metrics.md
│
├── output/
│   ├── config.schema.json          # JSON Schema for repo-intel.yml
│   ├── report.schema.json          # SARIF 2.1.0 + repo-intel extensions
│   ├── repo-intel.example.yml      # Annotated reference config
│   ├── .repo-intelignore.example   # Gitignore-style exclusions
│   ├── report-template.html        # HTML rendering of the SARIF document
│   └── report-template.md
│
├── examples/                       # Sample generated reports
│
└── .github/
    ├── workflows/ci.yml
    ├── dependabot.yml
    └── SECURITY.md
```

---

## Health scoring

| Score | Status | Meaning |
|-------|--------|---------|
| 8.0–10 | Good         | Production-ready, minor improvements only |
| 5.0–7.9 | Needs work | Functional but has gaps requiring attention |
| 3.0–4.9 | Poor       | Multiple significant issues affecting quality |
| 0.0–2.9 | Critical   | Immediate action required |

Overall score is a weighted mean of the six scoring dimensions. If a skill is skipped or fails, its weight is removed and the remainder is renormalised. See [skills/SCORING-CONTRACT.md](skills/SCORING-CONTRACT.md) §2.

---

## Outputs

Every run produces a SARIF 2.1.0 document as the canonical artefact. HTML and Markdown are derived renderings.

| File | Purpose |
|------|---------|
| `report.sarif` | Canonical. Validates against [output/report.schema.json](output/report.schema.json). Plugs into GitHub Code Scanning, Azure DevOps, Defender for DevOps, SonarQube, DefectDojo. |
| `report.html`  | Self-contained HTML. Includes Mermaid architecture diagrams. |
| `report.md`    | Markdown rendering for wikis or PR comments. |

Exit codes (when invoked from CI via `thresholds` in [repo-intel.yml](output/repo-intel.example.yml)):

| Code | Meaning |
|------|---------|
| 0 | No threshold breached |
| 1 | `fail_on` severity hit, or `min_scores` / `max_findings` violated |
| 2 | Analyser error (config invalid, skill crash, SARIF validation failed) |

---

## What it is not

For clarity, repo-intel today has **no** HTTP API, database, message queue, multi-tenancy, or cloud deployment. It runs in a local Claude Code session against a local checkout. Larger platform ideas live in [ROADMAP.md](ROADMAP.md).

---

## Add a custom skill

[skills/HOW-TO-ADD-SKILL.md](skills/HOW-TO-ADD-SKILL.md) — new skills are Markdown files. No code required. Your skill must declare its `RI-*` rules in [skills/FINDING-SCHEMA.md](skills/FINDING-SCHEMA.md) and return the [SUBAGENT-OUTPUT.md](skills/SUBAGENT-OUTPUT.md) envelope.

---

## Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md) and [.github/SECURITY.md](.github/SECURITY.md).
