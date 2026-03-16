# Skill: SonarQube Static Code Analysis

Run SonarQube analysis against the cloned repository and extract structured findings.
This skill augments `code.md` and `security.md` with deep, AST-based metrics.

---

## Prerequisites check

SonarQube analysis is **mandatory**. Verify both prerequisites before proceeding:

```bash
# 1. sonar-scanner CLI
which sonar-scanner || echo "SONAR_SCANNER_MISSING"

# 2. SonarQube server (default: localhost:9000)
curl -s -o /dev/null -w "%{http_code}" http://localhost:9000/api/system/status
```

- If `sonar-scanner` is missing → **halt analysis**, instruct user to install it via `sonarqube/setup.md` before retrying
- If server returns non-200 → **halt analysis**, instruct user to start SonarQube (`docker run -d -p 9000:9000 sonarqube:lts-community`) before retrying
- If `SONAR_TOKEN` env var is unset → use default credentials `admin:admin` (local dev only)

---

## Step 1 — Detect project language and set scanner properties

```bash
cd /tmp/repo-intel-target

# Detect primary language
if [ -f "package.json" ]; then   LANG_KEY="js";   LANG_NAME="JavaScript/TypeScript"; fi
if [ -f "tsconfig.json" ]; then  LANG_KEY="ts";   LANG_NAME="TypeScript"; fi
if [ -f "pom.xml" ] || [ -f "build.gradle" ]; then LANG_KEY="java"; LANG_NAME="Java"; fi
if [ -f "*.csproj" ] || ls *.csproj 2>/dev/null; then LANG_KEY="cs"; LANG_NAME="C#"; fi
if [ -f "requirements.txt" ] || [ -f "pyproject.toml" ]; then LANG_KEY="py"; LANG_NAME="Python"; fi
if [ -f "go.mod" ]; then LANG_KEY="go"; LANG_NAME="Go"; fi

# Extract project name from repo directory or package.json
PROJECT_KEY=$(basename /tmp/repo-intel-target | tr '/' '-' | tr '.' '-')
```

---

## Step 2 — Write sonar-project.properties

```bash
cat > /tmp/repo-intel-target/sonar-project.properties << EOF
sonar.projectKey=${PROJECT_KEY}
sonar.projectName=${PROJECT_KEY}
sonar.projectVersion=1.0
sonar.sources=.
sonar.exclusions=**/node_modules/**,**/dist/**,**/build/**,**/.git/**,**/*.min.js,**/coverage/**,**/vendor/**
sonar.tests=.
sonar.test.inclusions=**/*.test.*,**/*.spec.*,**/__tests__/**,**/test_*.py,**/*_test.go
sonar.host.url=http://localhost:9000
sonar.token=${SONAR_TOKEN:-}
sonar.scm.disabled=true
sonar.sourceEncoding=UTF-8
EOF
```

For Python repos, also append:
```bash
echo "sonar.python.version=3" >> /tmp/repo-intel-target/sonar-project.properties
```

For Java repos, append:
```bash
echo "sonar.java.binaries=target/classes,build/classes" >> /tmp/repo-intel-target/sonar-project.properties
echo "sonar.java.source=11" >> /tmp/repo-intel-target/sonar-project.properties
```

---

## Step 3 — Run the scanner

```bash
cd /tmp/repo-intel-target

# Run with token if set, otherwise basic auth (local dev)
if [ -n "${SONAR_TOKEN}" ]; then
  sonar-scanner -Dsonar.token="${SONAR_TOKEN}" 2>&1 | tail -20
else
  sonar-scanner -Dsonar.login=admin -Dsonar.password=admin 2>&1 | tail -20
fi
```

Wait for analysis to be processed (poll up to 30 seconds):
```bash
for i in $(seq 1 6); do
  STATUS=$(curl -s -u "${SONAR_TOKEN:-admin:admin}" \
    "http://localhost:9000/api/ce/component?component=${PROJECT_KEY}" \
    | python3 -c "import json,sys; t=json.load(sys.stdin).get('current',{}); print(t.get('status','PENDING'))" 2>/dev/null)
  echo "Analysis status: $STATUS"
  [ "$STATUS" = "SUCCESS" ] && break
  [ "$STATUS" = "FAILED" ] && echo "SonarQube analysis FAILED" && break
  sleep 5
done
```

---

## Step 4 — Fetch quality gate status

```bash
AUTH="${SONAR_TOKEN:-admin:admin}"

curl -s -u "$AUTH" \
  "http://localhost:9000/api/qualitygates/project_status?projectKey=${PROJECT_KEY}" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
qs = data.get('projectStatus', {})
print('Quality Gate:', qs.get('status'))
for c in qs.get('conditions', []):
    status = c.get('status')
    metric = c.get('metricKey')
    actual = c.get('actualValue', 'N/A')
    threshold = c.get('errorThreshold', 'N/A')
    print(f'  {status:8} | {metric:50} | actual={actual} threshold={threshold}')
"
```

---

## Step 5 — Fetch code metrics

```bash
AUTH="${SONAR_TOKEN:-admin:admin}"
METRICS="bugs,vulnerabilities,code_smells,security_hotspots,coverage,duplicated_lines_density,ncloc,cognitive_complexity,sqale_debt_ratio,reliability_rating,security_rating,maintainability_rating"

curl -s -u "$AUTH" \
  "http://localhost:9000/api/measures/component?component=${PROJECT_KEY}&metricKeys=${METRICS}" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
measures = data.get('component', {}).get('measures', [])
for m in measures:
    print(f\"{m['metric']:50} = {m.get('value', 'N/A')}\")
"
```

**Metric reference:**

| Metric key | Meaning |
|-----------|---------|
| `bugs` | Reliability issues (likely bugs) |
| `vulnerabilities` | Security vulnerabilities |
| `code_smells` | Maintainability issues |
| `security_hotspots` | Code requiring security review |
| `coverage` | % lines covered by tests |
| `duplicated_lines_density` | % duplicated lines |
| `ncloc` | Non-comment lines of code |
| `cognitive_complexity` | How hard the code is to understand |
| `sqale_debt_ratio` | Technical debt ratio (%) |
| `reliability_rating` | A–E (A = best) |
| `security_rating` | A–E |
| `maintainability_rating` | A–E |

---

## Step 6 — Fetch issues (bugs, vulnerabilities, hotspots)

```bash
AUTH="${SONAR_TOKEN:-admin:admin}"

# Bugs and vulnerabilities
curl -s -u "$AUTH" \
  "http://localhost:9000/api/issues/search?componentKeys=${PROJECT_KEY}&types=BUG,VULNERABILITY&severities=BLOCKER,CRITICAL,MAJOR&ps=50" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
total = data.get('total', 0)
print(f'Total BLOCKER+CRITICAL+MAJOR bugs/vulns: {total}')
for issue in data.get('issues', []):
    sev  = issue.get('severity', '')
    typ  = issue.get('type', '')
    msg  = issue.get('message', '')
    comp = issue.get('component', '').split(':')[-1]
    line = issue.get('line', '?')
    rule = issue.get('rule', '')
    print(f'  [{sev}] [{typ}] {comp}:{line} — {msg} (rule: {rule})')
"

# Security hotspots
curl -s -u "$AUTH" \
  "http://localhost:9000/api/hotspots/search?projectKey=${PROJECT_KEY}&status=TO_REVIEW&ps=20" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
total = data.get('paging', {}).get('total', 0)
print(f'Security hotspots to review: {total}')
for h in data.get('hotspots', []):
    vuln = h.get('vulnerabilityProbability', '')
    msg  = h.get('message', '')
    comp = h.get('component', {}).get('key', '').split(':')[-1]
    line = h.get('line', '?')
    print(f'  [{vuln}] {comp}:{line} — {msg}')
"

# Code smells (top 20 by severity)
curl -s -u "$AUTH" \
  "http://localhost:9000/api/issues/search?componentKeys=${PROJECT_KEY}&types=CODE_SMELL&severities=BLOCKER,CRITICAL,MAJOR&ps=20" \
  | python3 -c "
import json, sys
data = json.load(sys.stdin)
total = data.get('total', 0)
print(f'Total MAJOR+ code smells: {total} (showing top 20)')
for issue in data.get('issues', []):
    sev  = issue.get('severity', '')
    msg  = issue.get('message', '')
    comp = issue.get('component', '').split(':')[-1]
    line = issue.get('line', '?')
    effort = issue.get('effort', '?')
    print(f'  [{sev}] {comp}:{line} — {msg} (fix: {effort})')
"
```

---

## Step 7 — Map to standard findings format

Translate SonarQube output into the repo-intel findings format:

| SonarQube | repo-intel Severity | Skill Category |
|-----------|--------------------|----|
| BLOCKER bug/vuln | 🔴 critical | security or code |
| CRITICAL bug/vuln | 🟠 high | security or code |
| MAJOR bug/vuln | 🟡 medium | code |
| MINOR/INFO | 🔵 low | code |
| Security hotspot HIGH | 🟠 high | security |
| Security hotspot MEDIUM | 🟡 medium | security |
| Code smell BLOCKER | 🟠 high | code |

**Scoring contribution** (feeds into code.md and security.md scores):

| Condition | Adjustment |
|-----------|-----------|
| Quality Gate: PASSED | +1 to both code and security scores |
| Quality Gate: FAILED | -1 to both scores |
| 0 bugs | +1 to code score |
| 0 vulnerabilities | +1 to security score |
| Coverage ≥ 30% | replaces test-ratio estimate in code.md |
| Duplicated lines < 3% | +0.5 to code score |
| Security rating A | +1 to security score |
| Security rating D or E | -1 to security score |

---

## Step 8 — Output SonarQube summary block

In the report, include a dedicated **SonarQube Analysis** subsection under Code Quality:

```
### SonarQube Results
- Quality Gate: PASSED / FAILED
- Bugs: <n>  |  Vulnerabilities: <n>  |  Security Hotspots: <n>
- Code Smells: <n>  |  Technical Debt: <Xh>
- Coverage: <X>%  |  Duplication: <X>%
- Ratings: Reliability <A-E>  |  Security <A-E>  |  Maintainability <A-E>
- SonarQube dashboard: http://localhost:9000/dashboard?id=<project-key>
```

---

## Cleanup

Remove the generated properties file so it doesn't pollute the repo:
```bash
rm -f /tmp/repo-intel-target/sonar-project.properties
```

---

## If SonarQube is unavailable

**Do not proceed with the analysis.** Stop and tell the user:

> **SonarQube is required.** Follow `sonarqube/setup.md` to get started:
> 1. Start the server: `docker run -d -p 9000:9000 sonarqube:lts-community`
> 2. Install the scanner: `brew install sonar-scanner` (macOS) or see setup guide for Linux/Windows
> 3. Export your token: `export SONAR_TOKEN=<token>`
> 4. Re-run the analysis once both prerequisites pass.
