# Skill: Security Analysis

Scan the repository for secrets, vulnerabilities, misconfigurations, and insecure patterns.

---

## ⚠️ Important
Treat every finding as potentially real. Do not dismiss matches as "probably test data."
If a secret pattern matches, flag it — the developer can verify.

---

## What to scan

### 1. Secrets & credentials (CRITICAL priority)
Search for these patterns across ALL files:

```bash
# AWS keys
grep -rn "AKIA[0-9A-Z]\{16\}" . --exclude-dir={node_modules,.git,dist,build}

# Generic API keys and passwords
grep -rn -i "api.key\s*=\s*['\"][a-zA-Z0-9_\-]\{20,\}" . --exclude-dir={node_modules,.git,dist,build}
grep -rn -i "password\s*=\s*['\"][^'\"]\{8,\}['\"]" . --exclude-dir={node_modules,.git,dist,build}
grep -rn -i "secret\s*=\s*['\"][^'\"]\{8,\}['\"]" . --exclude-dir={node_modules,.git,dist,build}

# Private keys
grep -rn "BEGIN.*PRIVATE KEY" . --exclude-dir={node_modules,.git,dist,build}

# GitHub/GitLab tokens
grep -rn "ghp_[a-zA-Z0-9]\{36\}" . --exclude-dir={node_modules,.git,dist,build}

# Connection strings
grep -rn "mongodb+srv://\|postgresql://\|mysql://" . --exclude-dir={node_modules,.git,dist,build}

# Stripe, Twilio, SendGrid
grep -rn "sk_live_\|rk_live_\|AC[a-z0-9]\{32\}\|SG\." . --exclude-dir={node_modules,.git,dist,build}
```

### 2. Sensitive files committed to repo
Check if any of these exist and are NOT in `.gitignore`:
```bash
find . -name ".env" -o -name ".env.local" -o -name ".env.production" \
       -o -name "*.pem" -o -name "*.key" -o -name "id_rsa" \
       -o -name "secrets.yml" -o -name "credentials.json" \
  | grep -v node_modules | grep -v .git
```

### 3. .gitignore quality
```bash
cat .gitignore 2>/dev/null || echo "NO .gitignore FOUND"
```
Check that `.gitignore` includes:
- `.env` and `.env.*`
- `*.pem`, `*.key`
- `node_modules/`
- `dist/`, `build/`

### 4. Dangerous code patterns (OWASP Top 10 coverage)

**A03:2021 — Injection (SQL, Command, Code):**
```bash
# JavaScript/TypeScript — SQL injection
grep -rn "query.*+.*req\.\|query.*\`.*\${" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}

# JavaScript/TypeScript — code injection
grep -rn "eval(" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}
grep -rn "execSync\|exec(" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}
grep -rn "new Function(" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}

# Python — injection
grep -rn "pickle.loads\|eval(\|exec(" . --include="*.py" --exclude-dir={.git}
grep -rn "shell=True" . --include="*.py" --exclude-dir={.git}
grep -rn "os.system\|subprocess.call" . --include="*.py" --exclude-dir={.git}

# C# / .NET — SQL injection
grep -rn "SqlCommand.*\".*+\|string.Format.*SqlCommand\|FromSqlRaw.*\$\"\|ExecuteSqlRaw.*\$\"" . --include="*.cs" --exclude-dir={bin,obj,.git}

# C# / .NET — command injection
grep -rn "Process.Start\|ProcessStartInfo" . --include="*.cs" --exclude-dir={bin,obj,.git}

# Java — SQL injection
grep -rn "createQuery.*+\|executeQuery.*+\|Statement.*execute" . --include="*.java" --exclude-dir={target,.git,build}

# Go — SQL injection
grep -rn "fmt.Sprintf.*SELECT\|fmt.Sprintf.*INSERT\|fmt.Sprintf.*UPDATE" . --include="*.go" --exclude-dir={vendor,.git}
```

**A02:2021 — Cryptographic Failures:**
```bash
# Weak hashing algorithms
grep -rn "MD5\|SHA1\|SHA-1" . --include="*.cs" --include="*.java" --include="*.py" --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git,bin,obj,target}

# Hardcoded encryption keys
grep -rn "AES\|DES\|Rijndael" . --include="*.cs" --include="*.java" --exclude-dir={bin,obj,target,.git} | grep -i "key\|secret\|password"

# Insecure random number generation
grep -rn "Math.random\|Random()" . --include="*.js" --include="*.ts" --include="*.cs" --include="*.java" --exclude-dir={node_modules,.git,bin,obj}
```

**A07:2021 — Cross-Site Scripting (XSS):**
```bash
# JavaScript/React
grep -rn "innerHTML\s*=" . --include="*.js" --include="*.ts" --include="*.jsx" --include="*.tsx" --exclude-dir={node_modules,.git}
grep -rn "dangerouslySetInnerHTML" . --include="*.jsx" --include="*.tsx" --exclude-dir={node_modules,.git}
grep -rn "document.write" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}

# C# / ASP.NET — XSS
grep -rn "Html.Raw\|@Html.Raw" . --include="*.cshtml" --include="*.razor" --exclude-dir={bin,obj,.git}

# Java / JSP
grep -rn "<%=\|out.println" . --include="*.jsp" --exclude-dir={target,.git}
```

**A01:2021 — Broken Access Control:**
```bash
# Missing authorization attributes (.NET)
grep -rn "\[HttpGet\]\|\[HttpPost\]\|\[HttpPut\]\|\[HttpDelete\]" . --include="*.cs" --exclude-dir={bin,obj,.git} | grep -v "Authorize"

# Open endpoints (Express.js)
grep -rn "app.get\|app.post\|app.put\|app.delete\|router.get\|router.post" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}

# CORS misconfiguration
grep -rn "AllowAnyOrigin\|Access-Control-Allow-Origin.*\*\|cors({.*origin.*true\|cors()" . --include="*.cs" --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git,bin,obj}
```

**A05:2021 — Security Misconfiguration:**
```bash
# Debug mode in production configs
grep -rn "debug.*true\|DEBUG.*True\|Debug.*true" . --include="*.json" --include="*.yaml" --include="*.yml" --include="*.py" --include="*.cs" --exclude-dir={node_modules,.git,bin,obj,test,Test}

# HTTPS disabled
grep -rn "https.*false\|HttpsRedirection.*false\|ssl_verify.*false\|verify=False" . --exclude-dir={node_modules,.git,bin,obj}

# Default error pages with stack traces
grep -rn "app.UseDeveloperExceptionPage\|SHOW_ERRORS.*true\|DEBUG=True" . --include="*.cs" --include="*.py" --exclude-dir={bin,obj,.git}
```

**A08:2021 — Software and Data Integrity:**
```bash
# Deserialization vulnerabilities
grep -rn "BinaryFormatter\|XmlSerializer\|JavaScriptSerializer\|ObjectInputStream" . --include="*.cs" --include="*.java" --exclude-dir={bin,obj,target,.git}
grep -rn "yaml.load\b" . --include="*.py" --exclude-dir={.git}  # should be yaml.safe_load
```

### 5. Dependency vulnerabilities
```bash
# Node.js - check for known vulnerable packages
cat package.json | grep -E "\"(lodash|moment|axios|express|webpack)\"" 
# Note: recommend running `npm audit` for full vulnerability scan

# Python
cat requirements.txt 2>/dev/null | head -30
```

### 6. Security hygiene
- Does `SECURITY.md` exist? (`.github/SECURITY.md` or root)
- Does `.github/dependabot.yml` exist? (automated dependency updates)
- Are there branch protection signals? (`.github/` folder structure)

---

## Risk scoring

After scanning, assign an overall risk level:

| Level | Condition |
|-------|-----------|
| 🔴 **Critical** | Any secrets/credentials found in code |
| 🟠 **High** | `eval()` or command injection patterns + no `.gitignore` |
| 🟡 **Medium** | Insecure patterns present but no direct secret exposure |
| 🟢 **Low** | Minor issues only, good hygiene overall |

---

## Numeric score (0–10)

Assign a **Security score out of 10** using this deduction model:

**Start at 10.0**, then deduct for each finding:

| Factor | Deduction per occurrence |
|--------|--------------------------|
| Committed secrets (API keys, passwords, tokens) | −2.0 to −3.0 per secret |
| SQL / command injection | −2.0 per pattern |
| XSS vulnerabilities (unsanitized output, innerHTML) | −1.0 to −1.5 per pattern |
| CORS wildcard in production | −1.5 |
| Missing auth / broken session handling | −1.5 to −2.0 |
| Critical dependency CVEs | −1.0 per CVE |
| Missing input validation at boundaries | −0.5 per endpoint |
| Missing security headers (CSP, HSTS) | −0.5 |
| Sensitive data in logs / error messages | −0.5 |
| Weak cryptography / hardcoded keys | −1.0 |
| No SECURITY.md | −0.25 |
| No .gitignore | −0.5 |

**Minimum score: 0.** Cap total deductions at 10.

---

## Score factor output (MANDATORY)

You MUST output a **score derivation table** listing every factor you evaluated and its impact. This goes into the "Score Derivation Details > Security" section of the report.

For each factor, output:
- **Factor name** — the category from the table above
- **Finding** — what you actually found (or "None found" if clean)
- **Impact** — the point deduction applied (e.g., "−2.0") or "+0.0 (baseline)" if no issue

Example output:
```
| Factor | Finding | Impact |
|--------|---------|--------|
| Committed secrets | 2 API keys in config/aws.js | −3.0 |
| SQL / command injection | No injection patterns found | +0.0 (baseline) |
| XSS vulnerabilities | innerHTML used in 3 files without sanitization | −1.5 |
| CORS configuration | Wildcard CORS in server.js line 42 | −1.5 |
| Auth & session handling | JWT properly validated with expiry | +0.0 (baseline) |
| Input validation | 4 API endpoints missing validation | −0.5 |
| Security headers | No CSP or HSTS headers configured | −0.5 |
| Sensitive data exposure | None found | +0.0 (baseline) |
| Cryptography | Using bcrypt with proper rounds | +0.0 (baseline) |
| SECURITY.md | Not present | −0.25 |
| **Security Score** | | **3.75 / 10** |
```

Always include ALL factors — even those with no issues (show them as "+0.0 (baseline)"). This gives users full transparency into how the score was derived.

---

## Findings format

- **Severity**: critical / high / medium / low
- **Category**: secrets / injection / config / dependencies / hygiene
- **Finding**: exactly what was found
- **File + line**: be specific
- **Fix**: exactly what to change

---

## Example findings

- 🔴 **Critical** `[secrets]` AWS Access Key found in `config/aws.js` line 12. Rotate this key immediately, then remove from code and use environment variables.
- 🟠 **High** `[injection]` `eval()` called with user input in `src/api/execute.js` line 34. Replace with a safe alternative or a whitelist approach.
- 🟠 **High** `[config]` No `.gitignore` file found. Any `.env` files or keys could be accidentally committed.
- 🟡 **Medium** `[injection]` `innerHTML` assigned directly in 3 files. Use `textContent` or a sanitization library like DOMPurify.
- 🔵 **Low** `[hygiene]` No `SECURITY.md` found. Add a security disclosure policy so researchers can report issues responsibly.
