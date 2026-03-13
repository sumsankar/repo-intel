# Plugin / Skill Framework

## Overview

The Skill Framework is the extensibility backbone of the platform. It defines how analysis capabilities are packaged, registered, discovered, and executed.

There are two ways to create a skill:
1. **Prompt-based skill** (current) — a Markdown file in `skills/` read by Claude Code
2. **Programmatic skill** (Phase 2+) — a Python class implementing the `Skill` protocol

Both models are supported and can coexist.

---

## Prompt-Based Skills (Markdown)

This is the current implementation. Skills are defined as Markdown documents that instruct Claude Code what to look for and how to report findings.

### Skill File Structure

```markdown
# Skill: [Name]

[One sentence: what this skill analyzes]

---

## What to check

### 1. [First check]
[What to look for, why it matters]

Commands to run:
```bash
# shell commands
```

### 2. [Second check]
...

---

## Findings format

- **Severity**: critical / high / medium / low
- **Category**: [categories]
- **Finding**: what you found
- **File**: exact path
- **Fix**: concrete recommendation

---

## Example findings

- 🔴 **Critical** `[category]` ...
- 🟡 **Medium** `[category]` ...
```

### Registering a Prompt Skill

Add to `ANALYZE.md` Step 2:

```markdown
### Step 2 — Load your skills

- `skills/code.md`
- `skills/architecture.md`
- `skills/security.md`
- `skills/devops.md`
- `skills/dependency.md`    ← new
- `skills/governance.md`    ← new
- `skills/your-skill.md`    ← your custom skill
```

---

## Programmatic Skills (Python)

For Phase 2+ automated analysis, skills are Python classes:

### Skill Protocol

```python
# src/domain/skill.py

from typing import Protocol, runtime_checkable

@runtime_checkable
class Skill(Protocol):
    """
    A Skill analyzes one dimension of a repository.
    All skills must be stateless and independently testable.
    """

    @property
    def name(self) -> str:
        """Unique identifier: 'security', 'code', 'devops', etc."""
        ...

    @property
    def display_name(self) -> str:
        """Human-readable name for reports."""
        ...

    @property
    def description(self) -> str:
        """One-sentence description of what this skill measures."""
        ...

    @property
    def weight(self) -> float:
        """
        Contribution to overall score (0.0 to 1.0).
        All skill weights must sum to 1.0.
        """
        ...

    @property
    def version(self) -> str:
        """Semantic version of this skill's detection rules."""
        ...

    async def analyze(self, context: "RepoContext") -> "SkillResult":
        """
        Analyze the repository and return structured findings.

        Must:
        - Complete within 120 seconds
        - Not modify the repository
        - Not make external HTTP calls (except registered integrations)
        - Return partial results on error rather than raising
        """
        ...
```

### Base Skill Class

Extend `BaseSkill` for common behavior:

```python
# src/skills/base.py

class BaseSkill:
    """Base class providing default scoring and error handling."""

    SEVERITY_DEDUCTIONS = {
        "critical": 3.0,
        "high":     1.5,
        "medium":   0.5,
        "low":      0.1,
    }

    def score(self, findings: list[Finding]) -> int:
        total_deduction = sum(
            self.SEVERITY_DEDUCTIONS[f.severity]
            for f in findings
        )
        return max(0, min(10, round(10 - total_deduction)))

    def _make_finding(
        self,
        severity: str,
        category: str,
        title: str,
        detail: str,
        fix: str,
        rule_id: str,
        file_path: str | None = None,
        line_number: int | None = None,
    ) -> Finding:
        return Finding(
            id=generate_id(),
            severity=severity,
            category=category,
            title=title,
            detail=detail,
            fix=fix,
            rule_id=rule_id,
            file_path=file_path,
            line_number=line_number,
            skill=self.name,
        )
```

---

## Creating a New Programmatic Skill

### Step 1: Create the skill class

```python
# src/skills/performance_skill.py

from src.skills.base import BaseSkill
from src.domain.skill import SkillResult
from src.domain.models import RepoContext, Finding
import time

class PerformanceSkill(BaseSkill):
    name = "performance"
    display_name = "Performance"
    description = "Detects N+1 query patterns, missing indexes, and large bundle signals"
    weight = 0.05
    version = "1.0.0"

    async def analyze(self, context: RepoContext) -> SkillResult:
        start = time.monotonic()
        findings: list[Finding] = []
        metrics: dict = {}

        try:
            findings.extend(await self._detect_n_plus_one(context))
            findings.extend(await self._detect_large_imports(context))
            metrics["n_plus_one_count"] = sum(
                1 for f in findings if f.category == "n_plus_one"
            )
        except Exception as e:
            # Return partial results, don't propagate
            import structlog
            structlog.get_logger().warning(
                "performance_skill_partial_failure",
                error=str(e)
            )

        return SkillResult(
            skill_name=self.name,
            status="complete",
            findings=findings,
            metrics=metrics,
            score=self.score(findings),
            elapsed_ms=round((time.monotonic() - start) * 1000)
        )

    async def _detect_n_plus_one(self, context: RepoContext) -> list[Finding]:
        # Look for ORM calls inside loops
        findings = []
        for file in context.parsed_repo.files:
            if file.language not in ("python", "javascript", "typescript"):
                continue
            # Detection logic here...
        return findings
```

### Step 2: Register the skill

```python
# src/skills/registry.py

SKILL_REGISTRY: dict[str, type[Skill]] = {
    "code":         CodeSkill,
    "architecture": ArchitectureSkill,
    "security":     SecuritySkill,
    "devops":       DevOpsSkill,
    "dependency":   DependencySkill,
    "governance":   GovernanceSkill,
    "performance":  PerformanceSkill,  # ← add here
}

def load_skills(names: list[str]) -> list[Skill]:
    if names == ["all"]:
        names = list(SKILL_REGISTRY.keys())
    return [SKILL_REGISTRY[name]() for name in names if name in SKILL_REGISTRY]
```

### Step 3: Update skill weights

Ensure all weights sum to 1.0:

```python
SKILL_WEIGHTS = {
    "security":     0.28,  # reduced from 0.30
    "code":         0.23,  # reduced from 0.25
    "architecture": 0.20,
    "devops":       0.14,  # reduced from 0.15
    "dependency":   0.10,
    "performance":  0.05,  # new
}
assert abs(sum(SKILL_WEIGHTS.values()) - 1.0) < 0.001
```

### Step 4: Write tests

```python
# tests/unit/skills/test_performance_skill.py

@pytest.mark.asyncio
async def test_detects_orm_in_loop():
    context = make_repo_context(files={
        "api/views.py": """
def get_all_users():
    users = User.objects.all()
    for user in users:
        profile = Profile.objects.get(user=user)  # N+1
    return users
"""
    })
    skill = PerformanceSkill()
    result = await skill.analyze(context)

    n_plus_one = [f for f in result.findings if f.category == "n_plus_one"]
    assert len(n_plus_one) >= 1
    assert "api/views.py" in n_plus_one[0].file_path
```

---

## Skill Metadata and Discovery

Skills self-describe their metadata for the API and dashboard:

```python
# GET /api/v1/skills returns:
[
  {
    "name": "security",
    "display_name": "Security",
    "description": "Secrets, vulnerabilities, misconfigurations",
    "weight": 0.30,
    "version": "2.1.0",
    "categories": ["secrets", "injection", "config", "crypto", "hygiene"],
    "rule_count": 47
  },
  ...
]
```

---

## Skill Versioning

Each skill has a semantic version. When detection rules change:
- **Patch bump** (1.0.x): bug fix, no behavior change for passing repos
- **Minor bump** (1.x.0): new detection rules added (may produce new findings)
- **Major bump** (x.0.0): scoring model changed (scores will differ from previous runs)

Version is stored in the `AnalysisRun` node in Neo4j, enabling trend analysis that accounts for skill version changes.

---

## Skill Ideas Backlog

| Skill | What it checks | Priority |
|-------|---------------|---------|
| `performance` | N+1 queries, missing DB indexes, bundle size | High |
| `accessibility` | Missing ARIA, alt tags, color contrast | Medium |
| `api-design` | REST conventions, versioning, error consistency | Medium |
| `logging` | Structured logging, sensitive data in logs | Medium |
| `database` | Migration files, raw SQL, connection pooling | Medium |
| `i18n` | Hardcoded strings, missing translation keys | Low |
| `documentation` | JSDoc/docstring coverage, stale comments | Low |
| `license-compliance` | License compatibility with org policy | High |
| `sbom` | Software Bill of Materials generation | High |

---

## Related Documents

- [Module Design Guidelines](module-design-guidelines.md)
- [Coding Standards](coding-standards.md)
- [Analysis Engines](../docs/analysis-engines.md)
- [HOW-TO-ADD-SKILL](../skills/HOW-TO-ADD-SKILL.md)
