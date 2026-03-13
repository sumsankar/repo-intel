# Governance Agent

## Role

The Governance Agent is the final agent in the analysis pipeline. It evaluates all skill results against configured organizational policies, calculates composite scores, determines compliance status, and prepares the governance section of the analysis report.

**Persona:** Engineering Director or VP of Engineering who cares deeply about engineering excellence but is pragmatic about prioritization. Distinguishes between "must fix now" (blocking) and "should fix this quarter" (advisory).

---

## Responsibilities

| Responsibility | Output |
|---------------|--------|
| Evaluate all skill scores against policy thresholds | `compliance_result` |
| Calculate weighted overall score | `metrics.overall_score` |
| Calculate Risk Posture Index (RPI) | `metrics.rpi` |
| Identify blocking policy violations | `result.blocking_violations` |
| Identify advisory violations | `result.advisory_violations` |
| Assign compliance status | `result.is_compliant` |
| Generate governance summary for report | Report governance section |
| Compare with previous run (trend) | Score delta |

---

## Governance Protocol

### Step 1: Receive Skill Results

The governance agent receives the completed results from all skills:

```python
skill_results = {
    "code":         SkillResult(score=7, findings=[...], metrics={...}),
    "architecture": SkillResult(score=6, findings=[...], metrics={...}),
    "security":     SkillResult(score=2, findings=[...], metrics={...}),
    "devops":       SkillResult(score=3, findings=[...], metrics={...}),
    "dependency":   SkillResult(score=8, findings=[...], metrics={...}),
}
```

### Step 2: Calculate Weighted Overall Score

```
Weighted Score =
  security_score   × 0.30  +
  code_score       × 0.25  +
  architecture_score × 0.20 +
  devops_score     × 0.15  +
  dependency_score × 0.10

Example:
  2 × 0.30 = 0.60
  7 × 0.25 = 1.75
  6 × 0.20 = 1.20
  3 × 0.15 = 0.45
  8 × 0.10 = 0.80
  ─────────────────
  Total     = 4.80 / 10
```

### Step 3: Calculate Risk Posture Index

```python
def calculate_rpi(all_findings: list[Finding]) -> int:
    weights = {"critical": 25, "high": 10, "medium": 3, "low": 1}
    raw = sum(weights[f.severity] for f in all_findings)
    return max(0, round(100 - min(raw / 2, 100)))

# Example: 2 critical, 5 high, 8 medium, 6 low
# raw = 2×25 + 5×10 + 8×3 + 6×1 = 50 + 50 + 24 + 6 = 130
# rpi = max(0, 100 - 65) = 35  →  🔴 Critical Risk
```

### Step 4: Policy Evaluation

For each configured policy rule, evaluate the expression against flattened metrics:

```python
POLICY_RULES = [
    PolicyRule(
        id="SEC-001",
        name="No committed secrets",
        expression="security.critical_findings_by_category.secrets == 0",
        blocking=True,
        severity="critical"
    ),
    PolicyRule(
        id="CODE-001",
        name="Minimum test coverage",
        expression="code.metrics.test_ratio >= 0.10",
        blocking=False,
        severity="high"
    ),
    # ... more rules
]

def evaluate_policy(rule: PolicyRule, metrics: dict) -> bool:
    return eval_safe_expression(rule.expression, metrics)
```

**Safe expression evaluator** (prevents code injection):
```python
ALLOWED_OPS = {ast.Eq, ast.NotEq, ast.Lt, ast.LtE, ast.Gt, ast.GtE}

def eval_safe_expression(expr: str, context: dict) -> bool:
    tree = ast.parse(expr, mode='eval')
    # Validate only comparison operations, no function calls
    for node in ast.walk(tree):
        if isinstance(node, ast.Call):
            raise PolicyError(f"Function calls not allowed in policy: {expr}")
    return eval(compile(tree, '<policy>', 'eval'), {"__builtins__": {}}, context)
```

### Step 5: Determine Compliance Status

```python
blocking_failures = [r for r in evaluated if not r.passed and r.blocking]
advisory_failures = [r for r in evaluated if not r.passed and not r.blocking]

is_compliant = len(blocking_failures) == 0
```

### Step 6: Generate Governance Section

```markdown
## Governance Assessment

**Status:** ❌ NON-COMPLIANT
**Overall Score:** 4.8/10
**Risk Posture Index:** 35/100 🔴

### Blocking Violations (must fix before production deployment)
| ID | Policy | Recommendation |
|----|--------|---------------|
| SEC-001 | No committed secrets | Revoke Firebase key at serviceAccountKey.json |
| DEP-001 | No critical CVEs | Upgrade log4j from 2.14.1 to 2.17.1+ |

### Advisory Violations (should fix this sprint/quarter)
| ID | Policy | Recommendation |
|----|--------|---------------|
| CODE-001 | Minimum test coverage | Current: 3%, Target: 10% |
| DEVOPS-001 | CI/CD required | Add GitHub Actions workflow |
| CODE-002 | No extremely large files | Split Context.cs (5,766 lines) |

### Passed Policies
- ✅ ARCH-001 — Lock file required
- ✅ DEP-002 — No high CVEs above threshold
```

---

## Trend Analysis

The governance agent compares the current run with the previous run:

```python
@dataclass
class TrendData:
    previous_score: float | None
    current_score: float
    score_delta: float           # positive = improvement
    new_violations: list[str]    # policies that newly failed
    resolved_violations: list[str] # policies that now pass

def get_trend(current: GovernanceResult, previous: GovernanceResult | None) -> TrendData:
    if previous is None:
        return TrendData(None, current.overall_score, 0, [], [])

    new_violations = [
        r.policy_id for r in current.blocking_violations + current.advisory_violations
        if r.policy_id not in {v.policy_id for v in previous.blocking_violations + previous.advisory_violations}
    ]
    resolved = [
        r.policy_id for r in previous.blocking_violations + previous.advisory_violations
        if r.policy_id not in {v.policy_id for v in current.blocking_violations + current.advisory_violations}
    ]

    return TrendData(
        previous_score=previous.overall_score,
        current_score=current.overall_score,
        score_delta=current.overall_score - previous.overall_score,
        new_violations=new_violations,
        resolved_violations=resolved
    )
```

---

## Portfolio Governance View

For multi-repository analyses, the governance agent aggregates across repos:

```cypher
// Get portfolio governance summary
MATCH (repo:Repository)-[:HAS_RUN]->(run:AnalysisRun)
WHERE run.started_at > datetime() - duration('P30D')
WITH repo, run ORDER BY run.started_at DESC
WITH repo, head(collect(run)) AS latest_run
RETURN
  repo.url AS repository,
  latest_run.overall_score AS score,
  latest_run.is_compliant AS compliant,
  latest_run.critical_count AS critical_findings,
  latest_run.started_at AS last_analyzed
ORDER BY score ASC
```

---

## CI/CD Gate Integration

The governance agent result can be used to gate CI/CD pipelines:

```yaml
# GitHub Actions
- name: Check governance compliance
  run: |
    RESULT=$(curl -s -X POST $REPO_INTEL_URL/api/v1/governance/evaluate \
      -H "Authorization: Bearer $REPO_INTEL_TOKEN" \
      -d '{"analysis_id": "${{ env.ANALYSIS_ID }}", "policy": "strict"}')

    IS_COMPLIANT=$(echo $RESULT | jq '.is_compliant')
    SCORE=$(echo $RESULT | jq '.governance_score')

    echo "Compliance: $IS_COMPLIANT"
    echo "Score: $SCORE"

    if [ "$IS_COMPLIANT" = "false" ]; then
      echo "::error::Repository is non-compliant. Blocking deployment."
      echo $RESULT | jq '.blocking_violations[].name'
      exit 1
    fi
```

---

## Governance Notification

When a repository transitions to non-compliant, the governance agent triggers notifications:

```python
async def notify_on_compliance_change(
    current: GovernanceResult,
    previous: GovernanceResult | None,
    repo_url: str
) -> None:
    if previous and previous.is_compliant and not current.is_compliant:
        # Repo became non-compliant — alert immediately
        await notification_service.send_alert(
            title=f"⚠️ {repo_url} is now NON-COMPLIANT",
            body=f"New blocking violations: {[v.name for v in current.blocking_violations]}",
            severity="critical",
            channels=["slack", "email"]
        )
    elif not previous and not current.is_compliant:
        # First analysis, already non-compliant
        await notification_service.send_alert(
            title=f"🔴 {repo_url} — First analysis: NON-COMPLIANT",
            body=f"Score: {current.overall_score:.1f}/10",
            severity="high",
            channels=["slack"]
        )
```

---

## Related Documents

- [Governance Model](../docs/governance-model.md)
- [Analysis Engines](../docs/analysis-engines.md)
- [Skill: Governance](../skills/governance.md)
- [API Design](../docs/api-design.md)
