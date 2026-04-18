# Skill: Dependency & Supply Chain Analysis

Analyze the repository's external dependencies for known vulnerabilities, license risk, outdated packages, and supply chain hygiene.

---

## What to check

### 1. Dependency manifest detection

Identify **all** package managers in use across all projects:

```bash
# Detect package managers — scan the entire repo
find . -maxdepth 4 -type f \( \
  -name "package.json" -o -name "package-lock.json" -o -name "yarn.lock" -o -name "pnpm-lock.yaml" \
  -o -name "requirements.txt" -o -name "pyproject.toml" -o -name "poetry.lock" -o -name "Pipfile" \
  -o -name "go.mod" -o -name "go.sum" \
  -o -name "pom.xml" -o -name "build.gradle" -o -name "build.gradle.kts" \
  -o -name "*.csproj" -o -name "*.sln" -o -name "Directory.Packages.props" -o -name "packages.config" \
  -o -name "Gemfile" -o -name "Gemfile.lock" \
  -o -name "composer.json" -o -name "composer.lock" \
  -o -name "Cargo.toml" -o -name "Cargo.lock" \
  -o -name "pubspec.yaml" -o -name "mix.exs" \
  \) | grep -v node_modules | grep -v .git | grep -v bin/ | grep -v obj/ | grep -v target/

# For .NET — list all NuGet package references across all projects
find . -name "*.csproj" -exec grep -H "PackageReference" {} \; | grep -v bin/ | grep -v obj/

# For .NET — check for centralized package management
find . -name "Directory.Packages.props" -o -name "Directory.Build.props" | head -5
cat Directory.Packages.props 2>/dev/null

# For .NET — check for legacy packages.config
find . -name "packages.config" | head -5
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

### 4.1. .NET known vulnerable packages

```bash
# Check for known vulnerable NuGet packages
find . -name "*.csproj" -exec cat {} \; 2>/dev/null | python3 -c "
import sys, re
risky = {
    'System.Text.Json': 'CVEs in older versions — ensure >= 8.0.0',
    'Newtonsoft.Json': 'Deserialization attacks in old versions — ensure >= 13.0.3',
    'log4net': 'Multiple CVEs — ensure >= 2.0.15',
    'System.Drawing.Common': 'Security issues on non-Windows — ensure >= 8.0.0',
    'Microsoft.AspNetCore.Mvc': 'Ensure using latest LTS framework version',
    'System.IdentityModel.Tokens.Jwt': 'Algorithm confusion — ensure >= 7.0.0',
    'BouncyCastle': 'Timing attacks in old versions — check for updates',
}
for line in sys.stdin:
    for pkg, note in risky.items():
        if pkg.lower() in line.lower():
            version = re.search(r'Version=\"([^\"]+)\"', line)
            ver = version.group(1) if version else 'unknown'
            print(f'FOUND: {pkg} {ver} → {note}')
" 2>/dev/null

# Check for outdated .NET framework targets
find . -name "*.csproj" -exec grep -H "TargetFramework" {} \; | grep -E "net[0-9]\.[0-9]|netcoreapp|netstandard"
# Flag: netcoreapp3.1, net5.0, net6.0 (out of support); recommend net8.0+
```

### 4.2. Java known vulnerable packages

```bash
# Check for known vulnerable Java packages in pom.xml
cat pom.xml 2>/dev/null | python3 -c "
import sys
risky = {
    'log4j-core': 'Log4Shell (CVE-2021-44228) — ensure >= 2.17.1',
    'spring-boot': 'Spring4Shell — ensure >= 2.7.x or 3.x',
    'commons-collections': 'Deserialization attacks — ensure >= 3.2.2',
    'jackson-databind': 'Multiple CVEs — ensure >= 2.15.0',
    'snakeyaml': 'CVE-2022-1471 — ensure >= 2.0',
    'commons-text': 'Text4Shell — ensure >= 1.10.0',
    'hibernate-core': 'SQL injection in older versions — ensure >= 5.6.x',
}
for line in sys.stdin:
    for pkg, note in risky.items():
        if pkg.lower() in line.lower():
            print(f'FOUND: {pkg} → {note}')
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
    'node-sass':    'Deprecated — migrate to sass (dart-sass)',
    'create-react-class': 'Use ES6 classes or function components',
    'react-router-dom': 'If v5 — migrate to v6',
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

## Numeric score (0–10)

Assign a **Dependency Risk score out of 10** using this deduction model:

**Start at 10.0**, then deduct:

| Factor | Deduction |
|--------|-----------|
| Critical CVEs (per CVE, max −4.0) | −2.0 each |
| High CVEs (per CVE, max −3.0) | −0.5 each |
| Medium CVEs (aggregate, max −1.0) | −0.25 per 3 CVEs |
| Deprecated runtime dependencies (max −2.0) | −0.5 per package |
| GPL/AGPL licensed deps in non-GPL project (per dep) | −0.5 each |
| Unknown license dependencies (per dep, max −1.0) | −0.25 each |
| No lock file present | −1.0 |
| Significantly outdated deps (2+ major versions behind, max −1.0) | −0.25 per dep |
| Excessive dependency count (>150 direct) | −0.5 |
| Unpinned dependency versions (max −0.5) | −0.1 per unpinned |

**Minimum score: 0.**

---

## Score factor output (MANDATORY)

You MUST output a **score derivation table** listing every factor you evaluated and its impact. This goes into the "Score Derivation Details > Dependency Risk" section of the report.

For each factor, output:
- **Factor name** — the category from the table above
- **Finding** — what you found (counts, specific packages)
- **Impact** — the point deduction or "+0.0 (baseline)"

Example:
```
| Factor | Finding | Impact |
|--------|---------|--------|
| Critical CVEs | 0 critical vulnerabilities | +0.0 (baseline) |
| High CVEs | 3 high CVEs (lodash, express, jsonwebtoken) | −1.5 |
| Medium CVEs | 7 medium CVEs | −0.5 |
| Deprecated packages | moment@2.29.4, request@2.88.2 deprecated | −1.0 |
| License compliance | 1 GPL-3.0 transitive dependency (some-pkg) | −0.5 |
| Unknown licenses | 0 packages with unknown licenses | +0.0 (baseline) |
| Lock file | package-lock.json present and consistent | +0.0 (baseline) |
| Outdated dependencies | 4 deps are 2+ major versions behind | −1.0 |
| Dependency count | 87 direct dependencies (reasonable) | +0.0 (baseline) |
| Version pinning | All versions pinned in lock file | +0.0 (baseline) |
| **Dependency Risk Score** | | **5.5 / 10** |
```

Always include ALL factors even if no issues were found.

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

---

## Rules emitted

Follows [FINDING-SCHEMA.md](FINDING-SCHEMA.md) and [SCORING-CONTRACT.md](SCORING-CONTRACT.md).

| Rule ID | Default | Source |
|---------|---------|--------|
| RI-DEP-001-KNOWN-CVE-CRITICAL | critical | CVSS ≥ 9.0 from OSV / GHSA / NVD |
| RI-DEP-002-KNOWN-CVE-HIGH | high | CVSS 7.0–8.9 |
| RI-DEP-003-KNOWN-CVE-MEDIUM | medium | CVSS 4.0–6.9 |
| RI-DEP-004-DEPRECATED-PACKAGE | medium | Registry `deprecated` flag or maintainer advisory |
| RI-DEP-005-LICENSE-RISK | high | GPL/AGPL/SSPL/BUSL in production deps |
| RI-DEP-006-UNLICENSED | medium | No declared licence in package metadata |
| RI-DEP-007-OUTDATED-MAJOR | low | Installed version ≥ 2 majors behind latest stable |
| RI-DEP-008-DIRECT-TRANSITIVE-CONFLICT | medium | Lockfile contains conflicting versions of the same package |

**Source selection:** the CVE source is configured via `skill_config.dependency.severity_source` (default `osv`). Each finding MUST include the advisory ID in `properties.cve` (e.g. `CVE-2023-12345` or `GHSA-abcd-efgh-ijkl`).

**Return format:** [SUBAGENT-OUTPUT.md](SUBAGENT-OUTPUT.md).
