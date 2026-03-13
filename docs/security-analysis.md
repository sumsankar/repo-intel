# Security Analysis

## Overview

The Security Analysis engine is the highest-weighted component of the platform (30% of overall score). It scans repositories for secrets, credentials, vulnerable code patterns, configuration weaknesses, and security hygiene gaps.

---

## Analysis Categories

| Category | Examples | Default Severity |
|----------|---------|-----------------|
| `secrets` | API keys, passwords, private keys committed in code | Critical |
| `injection` | SQL injection, command injection, eval() patterns | High |
| `config` | debug mode on, CORS *, HTTP not HTTPS | High |
| `crypto` | MD5/SHA1 for passwords, non-CSPRNG for tokens | High |
| `data` | PII/PHI files committed, real data in test fixtures | High |
| `auth` | Hardcoded tokens, bypass mechanisms | High |
| `hygiene` | Missing SECURITY.md, Dependabot, branch protection | Low-Medium |

---

## Secrets Detection

### Pattern Library

The engine scans every file (except excluded dirs and binary files) for these patterns:

```python
SECRET_PATTERNS = [
    # Cloud providers
    Pattern(name="AWS Access Key",      regex=r"AKIA[0-9A-Z]{16}",          severity="critical"),
    Pattern(name="AWS Secret Key",      regex=r"(?i)aws.secret.{0,20}['\"][0-9a-zA-Z/+]{40}['\"]", severity="critical"),
    Pattern(name="GCP Service Account", regex=r'"type": "service_account"',  severity="critical"),
    Pattern(name="Azure Storage Key",   regex=r"AccountKey=[a-zA-Z0-9+/]{86}==", severity="critical"),

    # Version control tokens
    Pattern(name="GitHub PAT (classic)",regex=r"ghp_[a-zA-Z0-9]{36}",       severity="critical"),
    Pattern(name="GitHub PAT (fine)",   regex=r"github_pat_[a-zA-Z0-9_]{82}",severity="critical"),
    Pattern(name="GitLab Token",        regex=r"glpat-[a-zA-Z0-9\-]{20}",   severity="critical"),

    # Payment processors
    Pattern(name="Stripe Live Key",     regex=r"sk_live_[a-zA-Z0-9]{24,}",  severity="critical"),
    Pattern(name="Stripe Restricted",   regex=r"rk_live_[a-zA-Z0-9]{24,}",  severity="critical"),

    # Communication
    Pattern(name="Twilio Account SID",  regex=r"AC[a-z0-9]{32}",            severity="critical"),
    Pattern(name="SendGrid API Key",    regex=r"SG\.[a-zA-Z0-9\-_]{22}\.[a-zA-Z0-9\-_]{43}", severity="critical"),
    Pattern(name="Slack Bot Token",     regex=r"xoxb-[0-9]{11,12}-[0-9]{11,12}-[a-zA-Z0-9]{24}", severity="critical"),

    # Cryptographic material
    Pattern(name="RSA Private Key",     regex=r"-----BEGIN RSA PRIVATE KEY-----", severity="critical"),
    Pattern(name="EC Private Key",      regex=r"-----BEGIN EC PRIVATE KEY-----",  severity="critical"),
    Pattern(name="Generic Private Key", regex=r"-----BEGIN PRIVATE KEY-----",     severity="critical"),

    # Database connection strings
    Pattern(name="PostgreSQL URL",      regex=r"postgresql://[^:]+:[^@]+@",  severity="critical"),
    Pattern(name="MySQL URL",           regex=r"mysql://[^:]+:[^@]+@",       severity="critical"),
    Pattern(name="MongoDB Atlas",       regex=r"mongodb\+srv://[^:]+:[^@]+@",severity="critical"),
    Pattern(name="SQL Server",          regex=r"(?i)(password|pwd)\s*=\s*[A-Za-z0-9@#$%^&+=]{8,}", severity="high"),

    # Generic patterns (higher false positive rate)
    Pattern(name="Generic Password",    regex=r'(?i)password\s*[=:]\s*["\'][^"\']{8,}["\']', severity="high"),
    Pattern(name="Generic API Key",     regex=r'(?i)api.?key\s*[=:]\s*["\'][a-zA-Z0-9_\-]{20,}["\']', severity="high"),
    Pattern(name="Generic Secret",      regex=r'(?i)secret\s*[=:]\s*["\'][^"\']{8,}["\']', severity="high"),
    Pattern(name="JWT Secret",          regex=r'(?i)jwt.?secret\s*[=:]\s*["\'][^"\']{8,}["\']', severity="high"),
]
```

### Semgrep Integration

For deeper vulnerability scanning beyond regex, the engine runs Semgrep with the `p/security-audit` and `p/secrets` rulesets:

```bash
semgrep scan \
  --config p/security-audit \
  --config p/secrets \
  --config p/owasp-top-ten \
  --json \
  --output semgrep-results.json \
  /path/to/repo
```

---

## Vulnerable Code Patterns

### Injection Vulnerabilities

```python
INJECTION_PATTERNS = {
    "sql_injection_concat": {
        "pattern": r'(query|sql|execute)\s*[+,]\s*(req|request|params|body|input)',
        "languages": ["javascript", "typescript", "python"],
        "severity": "high",
        "fix": "Use parameterized queries or an ORM"
    },
    "command_injection": {
        "pattern": r'(exec|execSync|popen|system|subprocess\.call)\s*\(',
        "languages": ["javascript", "python"],
        "severity": "high",
        "fix": "Avoid shell=True; use subprocess with argument lists"
    },
    "eval_usage": {
        "pattern": r'\beval\s*\(',
        "languages": ["javascript", "typescript", "python"],
        "severity": "high",
        "fix": "Replace eval() with safer alternatives (JSON.parse, ast.literal_eval)"
    },
}
```

### Cryptographic Weaknesses

| Pattern | Why dangerous | Fix |
|---------|--------------|-----|
| `md5(password)` | MD5 is not a password hash — it's preimage-reversible | Use bcrypt, argon2, or PBKDF2 |
| `sha1(password)` | Same as above | Same fix |
| `Math.random()` for tokens | Predictable seed | Use `crypto.randomBytes()` |
| `System.Random` for OTPs | Clock-seeded, predictable | Use `RandomNumberGenerator.GetInt32()` |
| `base64(password)` | Trivially reversible | Use real password hashing |
| DES, RC4 encryption | Broken algorithms | Use AES-256-GCM |

---

## Sensitive File Detection

Files that should never be committed to source control:

```python
SENSITIVE_FILE_PATTERNS = [
    "*.pem", "*.key", "*.p12", "*.pfx", "*.cer",
    ".env", ".env.*", ".env.local", ".env.production",
    "id_rsa", "id_dsa", "id_ecdsa", "id_ed25519",
    "secrets.yml", "secrets.yaml",
    "credentials.json", "serviceAccountKey.json",
    "*.keystore", "google-services.json",
    "terraform.tfvars", "*.auto.tfvars",
    ".npmrc",  # may contain auth tokens
    ".pypirc", # may contain PyPI password
]
```

For each sensitive file found:
1. Check if it is in `.gitignore`
2. If NOT in `.gitignore` → Critical finding
3. If in `.gitignore` but file exists on disk → High finding (may already be in history)

---

## .gitignore Quality Assessment

A well-configured `.gitignore` is scored on whether it covers:

| Pattern | Importance |
|---------|-----------|
| `.env` and `.env.*` | Critical |
| `*.pem`, `*.key` | Critical |
| `node_modules/` | High |
| `dist/`, `build/` | Medium |
| `*.log` | Low |
| IDE folders (`.idea/`, `.vscode/`) | Low |

---

## Security Hygiene Checks

| Check | Pass Condition | Severity if failed |
|-------|---------------|--------------------|
| `SECURITY.md` exists | File at root or `.github/SECURITY.md` | Low |
| Dependabot configured | `.github/dependabot.yml` exists | Medium |
| Branch protection evident | `.github/` folder with CODEOWNERS or branch rules | Low |
| No `debug=true` in prod config | Not present or `false` | High |
| No `customErrors mode="Off"` | Not present in .NET web.config | High |
| HTTPS enforced | No `http://` in base URLs or CSP headers | Medium |
| Security headers in responses | X-Content-Type, X-Frame-Options, CSP | Medium |

---

## Risk Level Determination

```python
def determine_risk_level(findings: list[Finding]) -> str:
    critical = [f for f in findings if f.severity == "critical"]
    high      = [f for f in findings if f.severity == "high"]

    if any(f.category == "secrets" for f in critical):
        return "CRITICAL"  # Secrets exposed = always critical
    elif len(critical) > 0:
        return "CRITICAL"
    elif len(high) >= 3:
        return "HIGH"
    elif len(high) > 0:
        return "MEDIUM"
    else:
        return "LOW"
```

---

## Security Score Calculation

```python
def calculate_security_score(findings: list[Finding]) -> int:
    critical_count = sum(1 for f in findings if f.severity == "critical")
    high_count     = sum(1 for f in findings if f.severity == "high")
    medium_count   = sum(1 for f in findings if f.severity == "medium")

    # Start at 10, deduct per finding
    score = 10
    score -= critical_count * 3    # -3 per critical
    score -= high_count * 1.5      # -1.5 per high
    score -= medium_count * 0.5    # -0.5 per medium

    return max(0, min(10, round(score)))
```

---

## When Secrets Are Found: Immediate Guidance

The report always includes this escalation path when secrets are found:

1. **Revoke immediately** — follow provider-specific instructions (Firebase Console, AWS IAM, GitHub Settings)
2. **Rotate** — generate new credentials; store in a secrets manager (HashiCorp Vault, AWS Secrets Manager, Azure Key Vault)
3. **Purge from history** — `git filter-repo --path <file> --invert-paths`
4. **Update .gitignore** — add the file pattern
5. **Notify affected services** — assume the secret was already compromised

---

## Related Documents

- [Analysis Engines](analysis-engines.md)
- [Security Audit Agent](../agents/security-audit-agent.md)
- [Governance Model](governance-model.md)
