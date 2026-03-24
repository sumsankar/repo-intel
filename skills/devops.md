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
