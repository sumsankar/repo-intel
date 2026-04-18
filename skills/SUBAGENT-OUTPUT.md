# Subagent Output Contract

**Normative.** Every Phase-2 subagent (`code`, `architecture`, `devops`, `dependency`) and every directly-run skill (`security`, `governance`, `claude-metrics`) MUST return a single YAML document conforming to the schema below. The main agent in ANALYZE.md parses these into SARIF results and runs schema validation before synthesis.

---

## 1. Wire format

A skill returns one fenced YAML block and nothing else. Prose, commentary, or additional text outside the fence causes the main agent to mark the skill as failed.

````
```yaml
skill: security
status: ok
score: 7.5
duration_ms: 42100
metrics:
  files_scanned: 1842
  secrets_checked: 12
  entropy_threshold: 4.0
score_factors:
  - rule: RI-SEC-001-HARDCODED-SECRET
    severity: critical
    count: 1
    base_impact: 0.7
    total_deduction: 2.1
  - rule: RI-SEC-009-MISSING-GITIGNORE-SECRETS
    severity: medium
    count: 1
    base_impact: 0.3
    total_deduction: 0.3
findings:
  - ruleId: RI-SEC-001-HARDCODED-SECRET
    severity: critical
    file: src/config/aws.ts
    line: 42
    startColumn: 15
    endColumn: 35
    message: AWS access key committed at line 42. Rotate and move to AWS Secrets Manager.
    fix: Replace the literal with `process.env.AWS_ACCESS_KEY_ID` and rotate the key.
    cwe: CWE-798
    owasp: A07:2021
    evidence: "AKIA****************"
    partialFingerprint: a7b3c1e2
  - ruleId: RI-SEC-009-MISSING-GITIGNORE-SECRETS
    severity: medium
    file: .gitignore
    message: .env is not excluded — risk of future secret leaks.
    fix: Add `.env` and `.env.*` to .gitignore.
```
````

---

## 2. Required fields

| Field | Type | Description |
|-------|------|-------------|
| `skill`         | enum | One of `security`, `code`, `architecture`, `devops`, `dependency`, `governance`, `claude-metrics`. |
| `status`        | enum | `ok` \| `partial` \| `failed`. `partial` = ran but some checks skipped; the `notes` field explains. |
| `score`         | number | `[0.0, 10.0]`. Required unless `skill = claude-metrics` or `status = failed`. |
| `duration_ms`   | integer | Skill wall-clock time. |
| `score_factors` | array | Derivation table; sum of `total_deduction` plus `score` ≈ 10.0 (SCORING-CONTRACT.md §1). |
| `findings`      | array | Each element conforms to FINDING-SCHEMA.md §2. |

### Optional

| Field | Type | Description |
|-------|------|-------------|
| `metrics`       | object | Skill-specific measurements (LOC, coverage estimate, CVE count). Surfaced in the HTML report. |
| `notes`         | string | Human-readable explanation, especially when `status ≠ ok`. |
| `skipped_checks`| array  | List of `{ check, reason }` for transparency. |

---

## 3. Size limits

- **Total response size ≤ 8 KiB** per subagent. The main agent aborts parsing above this.
- **Findings per skill ≤ 500.** If more exist, subagent truncates lowest-severity first and sets `status: partial` with a note explaining.
- **Evidence string ≤ 120 characters**, HTML-escaped by the subagent (main agent re-escapes on splice).

---

## 4. Error envelope

If the skill cannot run at all:

```yaml
skill: dependency
status: failed
duration_ms: 120
notes: "osv-scanner not installed on runner"
```

The main agent sets the dimension score to `null`, records the skill in `repo-intel.skippedSkills`, and continues.

---

## 5. Validation

The main agent validates each subagent response against this schema and against `output/report.schema.json` rule IDs (every `ruleId` in `findings` must exist in FINDING-SCHEMA.md). Unknown rule IDs cause the finding to be dropped and a WARNING logged in the claude-metrics output.
