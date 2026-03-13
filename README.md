# AI Engineering Intelligence Platform

> Automated repository analysis for architecture insights, security findings, and engineering governance — powered by AI and static analysis.

---

## What is this?

**repo-intel** is an AI-powered platform that ingests software repositories and produces structured intelligence reports covering:

- **Code Quality** — complexity, test coverage, duplication, documentation
- **Architecture** — structure, dependencies, coupling, design patterns
- **Security** — secrets, vulnerabilities, misconfigurations, insecure patterns
- **DevOps** — CI/CD pipelines, containerization, IaC, repo hygiene
- **Dependency Risk** — supply chain vulnerabilities, license risk, outdated packages
- **Governance** — architecture compliance scoring, code health, risk posture

---

## Quick Start

### Analyze a repository

```
Follow ANALYZE.md and analyze https://github.com/org/repo
```

### Run specific skills only

```
Follow ANALYZE.md but only run the security and devops skills on https://github.com/org/repo
```

### Compare two repositories

```
Follow ANALYZE.md and compare https://github.com/org/repo-a vs https://github.com/org/repo-b
```

---

## Repository Structure

```
repo-intel/
├── README.md                        # This file
├── ANALYZE.md                       # Main orchestration prompt
│
├── skills/                          # Analysis skill modules
│   ├── code.md                      # Code quality analysis
│   ├── architecture.md              # Architecture & dependency analysis
│   ├── security.md                  # Security & secrets scanning
│   ├── devops.md                    # CI/CD & infrastructure analysis
│   ├── dependency.md                # Supply chain & license risk
│   ├── governance.md                # Architecture governance scoring
│   └── HOW-TO-ADD-SKILL.md          # Guide for adding custom skills
│
├── output/                          # Report templates
│   ├── report-template.md           # Markdown report template
│   └── report-template.html         # HTML report template
│
├── examples/                        # Sample generated reports
│   └── express-report-example.md
│
├── docs/                            # Platform documentation
│   ├── overview.md
│   ├── architecture.md
│   ├── system-design.md
│   ├── data-pipeline.md
│   ├── knowledge-graph.md
│   ├── ai-reasoning-engine.md
│   ├── analysis-engines.md
│   ├── repository-ingestion.md
│   ├── security-analysis.md
│   ├── devops-analysis.md
│   ├── governance-model.md
│   ├── scalability.md
│   ├── api-design.md
│   ├── deployment-architecture.md
│   └── development-roadmap.md
│
├── architecture/                    # Architecture design documents
│   ├── high-level-architecture.md
│   ├── component-architecture.md
│   ├── microservices-architecture.md
│   └── data-flow.md
│
├── engineering/                     # Engineering standards
│   ├── coding-standards.md
│   ├── module-design-guidelines.md
│   └── plugin-skill-framework.md
│
└── agents/                          # AI agent specifications
    ├── code-analyzer-agent.md
    ├── architecture-discovery-agent.md
    ├── security-audit-agent.md
    └── governance-agent.md
```

---

## Analysis Dimensions

| Dimension | What is measured | Skill |
|-----------|-----------------|-------|
| Code Quality | Complexity, test coverage, duplication, documentation | `skills/code.md` |
| Architecture | Structure, dependencies, coupling, patterns | `skills/architecture.md` |
| Security | Secrets, vulnerabilities, misconfigs | `skills/security.md` |
| DevOps | CI/CD, Docker, IaC, repo hygiene | `skills/devops.md` |
| Dependency Risk | CVEs, license risk, supply chain | `skills/dependency.md` |
| Governance | Compliance scoring, risk posture | `skills/governance.md` |

---

## Health Scoring

| Score | Status | Meaning |
|-------|--------|---------|
| 8–10 | 🟢 Good | Production-ready, minor improvements only |
| 5–7 | 🟡 Needs work | Functional but has gaps requiring attention |
| 3–4 | 🟠 Poor | Multiple significant issues affecting quality |
| 0–2 | 🔴 Critical | Immediate action required |

---

## Technology Stack

| Layer | Technology |
|-------|-----------|
| AI Engine | Claude (Anthropic) via Claude Code |
| Parsing | Tree-sitter (AST generation) |
| Security Scanning | Semgrep, Trivy |
| Static Analysis | SonarQube-compatible metrics |
| Graph Layer | Neo4j (knowledge graph) |
| API | FastAPI (Python) or ASP.NET Core |
| Frontend | React dashboard |

---

## Documentation Index

- [Platform Overview](docs/overview.md)
- [System Architecture](docs/architecture.md)
- [System Design](docs/system-design.md)
- [Data Pipeline](docs/data-pipeline.md)
- [Knowledge Graph](docs/knowledge-graph.md)
- [AI Reasoning Engine](docs/ai-reasoning-engine.md)
- [Analysis Engines](docs/analysis-engines.md)
- [Repository Ingestion](docs/repository-ingestion.md)
- [Security Analysis](docs/security-analysis.md)
- [DevOps Analysis](docs/devops-analysis.md)
- [Governance Model](docs/governance-model.md)
- [Scalability](docs/scalability.md)
- [API Design](docs/api-design.md)
- [Deployment Architecture](docs/deployment-architecture.md)
- [Development Roadmap](docs/development-roadmap.md)

---

## Adding Custom Skills

See [skills/HOW-TO-ADD-SKILL.md](skills/HOW-TO-ADD-SKILL.md) for a 5-minute guide to creating new analysis skills.

---

*AI Engineering Intelligence Platform — Built for engineering teams who take quality seriously.*
