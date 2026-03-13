# Skill: Engineering Governance

Evaluate the repository against organizational engineering standards and produce a governance compliance score and risk posture assessment.

---

## What to check

This skill runs LAST, after all other skills have completed. It synthesizes their results to produce a compliance verdict.

### 1. Aggregate scores from all completed skills

After running code, architecture, security, devops, and dependency skills, record:

| Dimension | Score (0–10) | Weight |
|-----------|-------------|--------|
| Security | ? | 30% |
| Code Quality | ? | 25% |
| Architecture | ? | 20% |
| DevOps | ? | 15% |
| Dependency Risk | ? | 10% |

**Weighted Overall Score:**
```
Overall = (security × 0.30) + (code × 0.25) + (architecture × 0.20) + (devops × 0.15) + (dependency × 0.10)
```

### 2. Policy evaluation

Evaluate each policy rule against the findings:

| Policy ID | Rule | Blocking |
|-----------|------|---------|
| GOV-SEC-001 | No critical secrets findings | YES |
| GOV-SEC-002 | Security score ≥ 5 | YES |
| GOV-DEP-001 | No critical CVEs | YES |
| GOV-CODE-001 | Test file ratio ≥ 10% | NO |
| GOV-CODE-002 | No files > 3000 lines | NO |
| GOV-DEV-001 | CI/CD pipeline exists | NO |
| GOV-DEV-002 | Tests run in CI | NO |
| GOV-ARCH-001 | Lock file exists | NO |

**Blocking policies:** a single failure = NON-COMPLIANT status

**Advisory policies:** failures are flagged but don't block compliance status

### 3. Compliance status determination

```
IF any blocking policy fails → Status: ❌ NON-COMPLIANT
ELSE IF any advisory policy fails → Status: ⚠️ CONDITIONALLY COMPLIANT
ELSE → Status: ✅ COMPLIANT
```

### 4. Risk Posture Index (RPI)

Calculate a 0–100 risk score based on all findings across all skills:

```
raw_score = (critical_count × 25) + (high_count × 10) + (medium_count × 3) + (low_count × 1)
RPI = max(0, 100 - raw_score / 2)
```

| RPI | Risk Level | Meaning |
|-----|-----------|---------|
| 80–100 | 🟢 Low Risk | Strong engineering practices |
| 60–79 | 🟡 Medium Risk | Some gaps, improvement roadmap clear |
| 40–59 | 🟠 High Risk | Multiple significant weaknesses |
| 0–39 | 🔴 Critical Risk | Immediate action required |

### 5. Architecture compliance check

Verify structural standards are met:
- Layered structure exists (controllers → services → repositories)
- No binary files committed (DLLs, JARs outside build tools)
- Naming conventions are consistent (check for typos like "Respository" vs "Repository")
- No `Archive/` or `Old/` directories committed (use git history instead)

### 6. Score trend (if prior analysis available)

If you have access to a previous analysis report, calculate:
```
Score delta = current_overall_score - previous_overall_score
New violations since last run: [list any policies that newly failed]
Resolved since last run: [list any policies that now pass]
```

---

## Governance report section format

Add a "Governance Assessment" section to the report with this structure:

```markdown
## Governance Assessment

**Status:** [✅ COMPLIANT / ⚠️ CONDITIONALLY COMPLIANT / ❌ NON-COMPLIANT]
**Overall Score:** [X.X/10] [🟢 / 🟡 / 🟠 / 🔴]
**Risk Posture Index:** [RPI/100] [🟢 / 🟡 / 🟠 / 🔴]

### Score Breakdown
| Dimension | Score | Weight | Contribution |
|-----------|-------|--------|-------------|
| Security | X/10 | 30% | X.XX |
| Code Quality | X/10 | 25% | X.XX |
| Architecture | X/10 | 20% | X.XX |
| DevOps | X/10 | 15% | X.XX |
| Dependency Risk | X/10 | 10% | X.XX |
| **Overall** | **X.X/10** | 100% | X.XX |

### Policy Compliance

**Blocking Violations (must fix before production):**
[List each blocking policy failure with specific finding reference]

**Advisory Violations (improve this sprint/quarter):**
[List each advisory failure with recommendation]

**Passed Policies:**
[List all passed policies]
```

---

## Findings format

- **Severity**: critical / high / medium / low
- **Category**: compliance / scoring / architecture / trend
- **Finding**: which policy failed and why
- **Fix**: specific action to pass the policy

---

## Example governance findings

- 🔴 **Critical** `[compliance]` GOV-SEC-001 FAILED: 2 critical security findings in `secrets` category. Repository is NON-COMPLIANT. **Fix:** Revoke and rotate all exposed credentials immediately.
- 🟠 **High** `[compliance]` GOV-CODE-001 FAILED: Test file ratio is 2% (target: 10%). **Fix:** Add unit tests for at least `UserService`, `AuthController`, and `PaymentProcessor`.
- 🟠 **High** `[compliance]` GOV-DEV-001 FAILED: No CI/CD pipeline detected. **Fix:** Add `.github/workflows/ci.yml` with build + test steps.
- 🟡 **Medium** `[scoring]` Overall score 3.8/10 is below the recommended threshold of 5.0. **Fix:** Address critical and high findings in security and devops dimensions first.
- 🟡 **Medium** `[architecture]` Typo in directory name: `Respository` (correct: `Repository`) propagated across 3 projects. **Fix:** Rename directories; update all import references.
- 🔵 **Low** `[trend]` Score decreased from 5.2 (last run) to 4.8 (this run). New violation: GOV-DEP-001 (new CVE found in log4j). **Fix:** Upgrade log4j as noted in dependency findings.

---

## Governance scoring summary table

Always end the governance section with this table:

```markdown
| Policy | Status | Impact |
|--------|--------|--------|
| No committed secrets | ❌ FAIL (BLOCKING) | Critical |
| Security score ≥ 5 | ❌ FAIL (BLOCKING) | High |
| No critical CVEs | ✅ PASS | — |
| Test coverage ≥ 10% | ❌ FAIL (advisory) | High |
| CI/CD pipeline | ❌ FAIL (advisory) | High |
| Lock file exists | ✅ PASS | — |
| No files > 3000 lines | ✅ PASS | — |

**Overall: 2/7 policies passed (29%) — ❌ NON-COMPLIANT**
```
