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

### 4. Dangerous code patterns
```bash
# JavaScript/TypeScript
grep -rn "eval(" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}
grep -rn "innerHTML\s*=" . --include="*.js" --include="*.ts" --include="*.jsx" --include="*.tsx" --exclude-dir={node_modules,.git}
grep -rn "dangerouslySetInnerHTML" . --include="*.jsx" --include="*.tsx" --exclude-dir={node_modules,.git}
grep -rn "execSync\|exec(" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}

# Python
grep -rn "pickle.loads\|eval(\|exec(" . --include="*.py" --exclude-dir={.git}
grep -rn "shell=True" . --include="*.py" --exclude-dir={.git}

# SQL injection signals
grep -rn "query.*+.*req\.\|query.*\`.*\${" . --include="*.js" --include="*.ts" --exclude-dir={node_modules,.git}
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
