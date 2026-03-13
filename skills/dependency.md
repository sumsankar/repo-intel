# Skill: Dependency & Supply Chain Analysis

Analyze the repository's external dependencies for known vulnerabilities, license risk, outdated packages, and supply chain hygiene.

---

## What to check

### 1. Dependency manifest detection

Identify what package managers are in use:

```bash
# Detect package managers
ls package.json package-lock.json yarn.lock pnpm-lock.yaml \
   requirements.txt pyproject.toml poetry.lock Pipfile Pipfile.lock \
   go.mod go.sum \
   pom.xml build.gradle \
   *.csproj *.sln \
   Gemfile Gemfile.lock \
   composer.json composer.lock \
   Cargo.toml Cargo.lock \
   2>/dev/null
```

### 2. Lock file presence (CRITICAL for supply chain)

A missing lock file means:
- `npm install` may install a different version than what was tested
- Security patches may or may not apply
- Supply chain attacks are easier (attacker bumps a transitive dependency)

```bash
# Node.js: check lock file
ls package-lock.json yarn.lock pnpm-lock.yaml 2>/dev/null || echo "NO LOCK FILE"

# Python: check for pinned versions
cat requirements.txt 2>/dev/null | grep -v "==" | grep -v "^#" | grep -v "^$"
# Any line without == is unpinned → medium finding
```

### 3. Dependency counts and bloat

```bash
# Node.js
cat package.json 2>/dev/null | python3 -c "
import json, sys
p = json.load(sys.stdin)
direct = len(p.get('dependencies', {}))
dev = len(p.get('devDependencies', {}))
print(f'Direct deps: {direct}')
print(f'Dev deps: {dev}')
if direct > 50:
    print('WARNING: High direct dependency count (>50) — bloat risk')
" 2>/dev/null

# Python
wc -l requirements.txt 2>/dev/null
cat pyproject.toml 2>/dev/null | grep -c "^[a-zA-Z]" 2>/dev/null
```

### 4. Known vulnerable packages (manual check)

Check for packages with well-known historical vulnerabilities:

```bash
# Node.js — check for historically problematic packages
cat package.json 2>/dev/null | python3 -c "
import json, sys
p = json.load(sys.stdin)
deps = {**p.get('dependencies',{}), **p.get('devDependencies',{})}

# Known vulnerable packages (illustrative — not exhaustive)
risky = {
    'lodash': 'Prototype pollution (CVE-2019-10744) — ensure >= 4.17.21',
    'minimist': 'Prototype pollution — ensure >= 1.2.6',
    'axios': 'SSRF risk in old versions — ensure >= 1.6.0',
    'moment': 'Path traversal in old versions + unmaintained — migrate to date-fns or dayjs',
    'log4j': 'Log4Shell (CVE-2021-44228) — ensure >= 2.17.1',
    'jsonwebtoken': 'Algorithm confusion — ensure >= 9.0.0',
    'node-fetch': 'Exposure in old versions — ensure >= 3.0.0',
}
for pkg, note in risky.items():
    if pkg in deps:
        print(f'FOUND: {pkg} {deps[pkg]} → {note}')
" 2>/dev/null

# Python — check for historically problematic packages
cat requirements.txt 2>/dev/null | python3 -c "
import sys
risky = {
    'pyyaml': 'Arbitrary code execution in old versions — ensure >= 5.4',
    'pillow': 'Multiple CVEs — ensure >= 10.0.0',
    'cryptography': 'Memory safety issues — ensure >= 41.0.0',
    'requests': 'Certificate verification bypass in old versions — ensure >= 2.31.0',
    'urllib3': 'SSRF/header injection — ensure >= 2.0.0',
    'django': 'Multiple CVEs — ensure >= 4.2 (LTS)',
    'flask': 'Multiple CVEs — ensure >= 3.0',
}
for line in sys.stdin:
    pkg = line.split('==')[0].split('>=')[0].strip().lower()
    if pkg in risky:
        print(f'FOUND: {line.strip()} → {risky[pkg]}')
" 2>/dev/null
```

### 5. Deprecated / unmaintained packages

```bash
# Node.js — well-known deprecated packages
cat package.json 2>/dev/null | python3 -c "
import json, sys
p = json.load(sys.stdin)
deps = {**p.get('dependencies',{}), **p.get('devDependencies',{})}

deprecated = {
    'moment':       'Unmaintained — migrate to date-fns or dayjs',
    'request':      'Deprecated by maintainer — migrate to axios or node-fetch',
    'node-uuid':    'Use uuid package instead',
    'bower':        'Deprecated — use npm directly',
    'tslint':       'Deprecated — migrate to ESLint',
    'cz-conventional-changelog': 'Check if maintained',
    'istanbul':     'Deprecated — use nyc or c8',
}
for pkg, note in deprecated.items():
    if pkg in deps:
        print(f'DEPRECATED: {pkg} → {note}')
" 2>/dev/null
```

### 6. License risk

```bash
# List licenses from package.json
cat package.json 2>/dev/null | python3 -c "
import json, sys
p = json.load(sys.stdin)
deps = p.get('dependencies', {})
print('Dependencies to check licenses for:')
for pkg, ver in list(deps.items())[:20]:
    print(f'  {pkg}: {ver}')
print('Run: npx license-checker --production --summary')
" 2>/dev/null

# Python
cat requirements.txt 2>/dev/null | python3 -c "
print('Packages to check licenses for:')
import sys
for line in sys.stdin:
    if line.strip() and not line.startswith('#'):
        print(f'  {line.strip()}')
print('Run: pip-licenses --with-urls')
" 2>/dev/null
```

**License risk classification:**

| License | Risk | Implication |
|---------|------|-------------|
| MIT, Apache-2.0, BSD | 🟢 Low | Safe for commercial use |
| LGPL | 🟡 Medium | Dynamic linking OK; review static linking |
| GPL, AGPL | 🔴 High | May require source disclosure |
| Unknown / No license | 🔴 High | Legally "all rights reserved" |

### 7. Trivy integration (when available)

```bash
# Run Trivy for CVE scanning (if installed)
trivy fs --security-checks vuln \
  --format json \
  --output /tmp/trivy-results.json \
  . 2>/dev/null

# Parse results
cat /tmp/trivy-results.json 2>/dev/null | python3 -c "
import json, sys
data = json.load(sys.stdin)
for result in data.get('Results', []):
    for vuln in result.get('Vulnerabilities', []):
        print(f'{vuln[\"Severity\"]} | {vuln[\"PkgName\"]} {vuln[\"InstalledVersion\"]} | {vuln[\"VulnerabilityID\"]} | Fix: {vuln.get(\"FixedVersion\", \"no fix\")}')
" 2>/dev/null || echo "Trivy not available — manual dependency review performed"
```

### 8. Security hygiene (Dependabot)

```bash
# Check for Dependabot configuration
cat .github/dependabot.yml 2>/dev/null || echo "No Dependabot configured"
```

An ideal Dependabot config:
```yaml
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    groups:
      dependencies:
        patterns: ["*"]
```

---

## Findings format

- **Severity**: critical / high / medium / low
- **Category**: cve / license / staleness / hygiene / lockfile
- **Finding**: specific package, version, CVE ID or license
- **Fix**: exact upgrade path or action

---

## Example findings

- 🟠 **High** `[cve]` `lodash@4.17.11` — CVE-2019-10744 (prototype pollution, CVSS 9.1). **Fix:** Upgrade to `lodash@4.17.21`.
- 🟠 **High** `[lockfile]` No `package-lock.json` found. Supply chain attacks are possible via transitive dependency manipulation. **Fix:** Run `npm install` and commit `package-lock.json`.
- 🟠 **High** `[cve]` `log4j-core@2.14.1` — CVE-2021-44228 (Log4Shell, CVSS 10.0 — RCE). **Fix:** Upgrade to `2.17.1` immediately.
- 🟡 **Medium** `[staleness]` `moment@2.29.4` — package is deprecated by its maintainers. **Fix:** Migrate to `date-fns` or `dayjs`.
- 🟡 **Medium** `[license]` `some-package@1.0.0` has GPL-3.0 license — may require source disclosure. **Fix:** Review with legal team or replace with a permissively-licensed alternative.
- 🟡 **Medium** `[hygiene]` No Dependabot configuration. Vulnerable dependencies are not automatically flagged. **Fix:** Add `.github/dependabot.yml`.
- 🔵 **Low** `[staleness]` `request@2.88.2` — deprecated by maintainer. **Fix:** Migrate to `axios` or native `fetch`.
- 🔵 **Low** `[lockfile]` Python `requirements.txt` has 3 unpinned dependencies (no version specifier). **Fix:** Pin all versions with `pip freeze > requirements.txt`.
