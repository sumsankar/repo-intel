# Skill: DevOps Analysis

Analyze CI/CD pipelines, containerization, infrastructure as code, and repo hygiene.

---

## What to check

### 1. CI/CD pipeline
Check for **all** pipeline configuration files:

```bash
# GitHub Actions
ls .github/workflows/ 2>/dev/null
cat .github/workflows/*.yml 2>/dev/null

# GitLab CI
cat .gitlab-ci.yml 2>/dev/null | head -50

# Azure DevOps Pipelines
cat azure-pipelines.yml 2>/dev/null
find . -name "*.yml" -path "*pipelines*" -o -name "*.yaml" -path "*pipelines*" 2>/dev/null | head -5

# Jenkins
cat Jenkinsfile 2>/dev/null

# Other CI
ls .circleci/config.yml .travis.yml buildkite.yml bitbucket-pipelines.yml 2>/dev/null
```

**If GitHub Actions found**, inspect each workflow:
```bash
cat .github/workflows/*.yml
```
Check each workflow for:
- ✅ Test step (`test`, `jest`, `pytest`, `go test`, `dotnet test`, `mvn test`)
- ✅ Lint step (`lint`, `eslint`, `flake8`, `golint`, `dotnet format`)
- ✅ Build step (`build`, `dotnet build`, `mvn package`, `go build`)
- ✅ Security scan (`codeql`, `snyk`, `trivy`, `semgrep`, `dotnet-security-scan`)
- ✅ Deploy/release step
- ✅ Artifact publishing (NuGet, npm, Docker registry)
- ⚠️ Hardcoded secrets (should use `${{ secrets.NAME }}` not literal values)
- ⚠️ Using `actions/checkout@v1` or `v2` (flag as outdated, recommend v4)
- ⚠️ Using `pull_request_target` without careful input sanitization

**If Azure DevOps Pipelines found**, check:
```bash
cat azure-pipelines.yml 2>/dev/null
```
- ✅ Pipeline uses templates (not inline scripts for everything)
- ✅ Uses `checkout: self` with `fetchDepth: 1` for performance
- ✅ Uses service connections (not hardcoded credentials)
- ⚠️ Uses deprecated task versions
- ⚠️ Missing `pool` specification (should pin vmImage version)

**If no CI found**: flag as high severity — no automated quality gates.

### 2. Docker
```bash
# Check for Dockerfiles
find . -name "Dockerfile*" | grep -v node_modules

# Check compose files
ls docker-compose.yml docker-compose.yaml docker-compose.*.yml 2>/dev/null

# Inspect Dockerfile if found
cat Dockerfile 2>/dev/null
```

**Dockerfile best practices to check:**
- ❌ `FROM image:latest` → should pin a specific version
- ❌ Running as root (no `USER` instruction) → add non-root user
- ❌ `ADD` used instead of `COPY` (unless intentional)
- ❌ `npm install` without `--production` in prod stage
- ✅ Multi-stage build (reduces image size)
- ✅ `.dockerignore` exists

### 3. Infrastructure as Code
```bash
# Terraform
find . -name "*.tf" | grep -v node_modules | head -5

# AWS CDK
ls cdk.json cdk.context.json 2>/dev/null

# Pulumi
ls Pulumi.yaml 2>/dev/null

# Ansible
ls ansible.cfg playbook.yml 2>/dev/null

# Kubernetes manifests
find . -name "*.yaml" -o -name "*.yml" | grep -v node_modules | \
  xargs grep -l "apiVersion:" 2>/dev/null | head -5

# Helm charts
ls charts/ helm/ 2>/dev/null
```

### 4. Repo hygiene checklist
```bash
ls README.md README LICENSE LICENSE.md CHANGELOG.md CONTRIBUTING.md \
   .editorconfig .prettierrc .eslintrc* .nvmrc .node-version 2>/dev/null
```

| File | Why it matters |
|------|---------------|
| `README.md` | First thing developers see — is it useful? |
| `LICENSE` | Required for open source use |
| `CHANGELOG.md` | Tracks what changed and when |
| `CONTRIBUTING.md` | Onboards new contributors |
| `.editorconfig` | Consistent formatting across editors |
| Lock file | Reproducible installs |

**README quality check** (if it exists):
- Does it have installation instructions?
- Does it have usage examples?
- Does it have a badge for CI status?
- Is it over 20 lines (substantive)?

### 5. Release & versioning
```bash
# Check for semantic versioning setup
cat package.json 2>/dev/null | grep '"version"'
ls CHANGELOG.md .releaserc* release.config.js 2>/dev/null

# Git tags (version history)
git tag | tail -10
```

---

## Numeric score (0–10)

Assign a **DevOps score out of 10** using this deduction model:

**Start at 10.0**, then deduct for missing or misconfigured items:

| Factor | Deduction if missing/failing |
|--------|------------------------------|
| No CI/CD pipeline at all | −3.0 |
| CI/CD exists but no test step | −1.5 |
| No security scanning in CI (SAST, dependency audit) | −1.5 |
| No Docker (when project type warrants it) | −0.5 |
| Docker images not pinned (using :latest) | −0.5 |
| Docker runs as root (no USER instruction) | −0.5 |
| No lock file (package-lock.json, yarn.lock, etc.) | −1.0 |
| No README or README is a stub | −0.5 |
| No LICENSE file | −0.5 |
| No CHANGELOG | −0.25 |
| No SECURITY.md | −0.25 |
| No Dependabot / Renovate config | −0.5 |
| No .editorconfig or formatting config | −0.25 |
| No IaC (when infrastructure is present) | −0.5 |

**Minimum score: 0.**

---

## Score factor output (MANDATORY)

You MUST output a **score derivation table** listing every factor you evaluated and its impact. This goes into the "Score Derivation Details > DevOps" section of the report.

For each factor, output:
- **Factor name** — the category from the table above
- **Finding** — what you found (present/absent, details)
- **Impact** — the point deduction or "+0.0 (baseline)"

Example:
```
| Factor | Finding | Impact |
|--------|---------|--------|
| CI/CD pipeline | GitHub Actions configured with build + test + deploy | +0.0 (baseline) |
| Tests in CI | Unit tests run on every PR | +0.0 (baseline) |
| Security scanning in CI | No SAST or dependency audit step | −1.5 |
| Docker | Dockerfile present with multi-stage build | +0.0 (baseline) |
| Docker image pinning | Using node:latest in Dockerfile | −0.5 |
| Docker non-root user | No USER instruction | −0.5 |
| Lock file | package-lock.json present | +0.0 (baseline) |
| README | Comprehensive with usage examples | +0.0 (baseline) |
| LICENSE | MIT license present | +0.0 (baseline) |
| CHANGELOG | Not present | −0.25 |
| SECURITY.md | Not present | −0.25 |
| Dependabot / Renovate | dependabot.yml configured | +0.0 (baseline) |
| IaC | Not applicable (no infrastructure) | +0.0 (N/A) |
| **DevOps Score** | | **7.0 / 10** |
```

Always include ALL factors even if no issues were found. Use "+0.0 (N/A)" for factors that don't apply to the project type.

---

## Findings format

- **Severity**: critical / high / medium / low
- **Category**: ci / docker / iac / hygiene / release
- **Finding**: specific observation
- **File**: where applicable
- **Fix**: what to do

---

## Example findings

- 🟠 **High** `[ci]` No CI/CD pipeline detected. Add GitHub Actions to automate testing on every pull request.
- 🟠 **High** `[ci]` GitHub Actions workflow has no test step. Tests are never automatically run — bugs reach production undetected.
- 🟡 **Medium** `[docker]` `Dockerfile` uses `FROM node:latest`. Pin to `FROM node:20-alpine` for reproducible builds.
- 🟡 **Medium** `[docker]` Container runs as root (no `USER` instruction). Add `USER node` before `CMD`.
- 🟡 **Medium** `[hygiene]` No `CHANGELOG.md`. Consider using conventional commits + `release-please` to automate release notes.
- 🔵 **Low** `[ci]` `actions/checkout@v2` used in 2 workflows. Upgrade to `@v4` for security and performance.
- 🔵 **Low** `[hygiene]` `README.md` has no usage examples. Add a quick start section.
