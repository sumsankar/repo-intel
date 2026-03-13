# Coding Standards

## Overview

These standards apply to all code written for the AI Engineering Intelligence Platform. They are enforced by automated linting and CI checks.

---

## Python (Backend)

### Style & Formatting
- **Formatter:** `ruff format` (replaces black)
- **Linter:** `ruff check`
- **Type checking:** `mypy --strict`
- **Line length:** 100 characters
- **Python version:** 3.12+

### Type Annotations

All function signatures must have full type annotations:

```python
# ✅ Good
async def analyze(
    self,
    context: RepoContext,
    timeout: int = 120
) -> SkillResult:
    ...

# ❌ Bad
async def analyze(self, context, timeout=120):
    ...
```

### Dataclasses and Pydantic

Prefer `@dataclass` for internal data structures. Use `pydantic.BaseModel` for API request/response models:

```python
# Internal data
@dataclass
class Finding:
    id: str
    severity: Literal["critical", "high", "medium", "low"]
    category: str
    file_path: str | None = None

# API model
class CreateAnalysisRequest(BaseModel):
    repo_url: HttpUrl
    skills: list[str] = Field(default=["all"])
    branch: str | None = None
```

### Async / Await

The entire backend is async. Rules:
- Use `async def` for all I/O-bound functions
- Use `asyncio.gather()` for concurrent tasks
- Never use `time.sleep()` — use `asyncio.sleep()`
- Never call sync blocking functions from async context — use `asyncio.run_in_executor()`

```python
# ✅ Good: parallel execution
results = await asyncio.gather(
    skill_code.analyze(ctx),
    skill_security.analyze(ctx),
    skill_devops.analyze(ctx),
)

# ❌ Bad: sequential when parallel is possible
result1 = await skill_code.analyze(ctx)
result2 = await skill_security.analyze(ctx)
result3 = await skill_devops.analyze(ctx)
```

### Error Handling

- Never catch `Exception` silently — always log
- Raise domain-specific exceptions, not generic ones
- Use `Result` types for expected failure states

```python
# ✅ Good
try:
    result = await clone_repo(url, dest)
except CloneAuthError as e:
    logger.warning("Clone auth failed", repo_url=url, error=str(e))
    raise
except CloneNotFoundError as e:
    raise AnalysisError(f"Repository not found: {url}") from e

# ❌ Bad
try:
    result = await clone_repo(url, dest)
except Exception:
    pass  # silent swallow
```

### Logging

Use structured logging with `structlog`:

```python
import structlog
logger = structlog.get_logger()

# ✅ Good: structured fields
logger.info(
    "analysis_complete",
    analysis_id=run_id,
    repo_url=repo_url,
    duration_ms=elapsed,
    score=overall_score,
    critical_count=critical_count
)

# ❌ Bad: unstructured string
logger.info(f"Analysis {run_id} done in {elapsed}ms with score {overall_score}")
```

### Testing

- **Framework:** `pytest` + `pytest-asyncio`
- **Mocking:** `unittest.mock` / `pytest-mock`
- **Coverage:** minimum 70% for new code
- **Structure:** `tests/` mirrors `src/` structure

```python
# tests/skills/test_security_skill.py
@pytest.mark.asyncio
async def test_detects_aws_key():
    context = make_repo_context(files={
        "config/aws.py": 'AWS_KEY = "AKIAIOSFODNN7EXAMPLE"'
    })
    skill = SecuritySkill()
    result = await skill.analyze(context)

    critical = [f for f in result.findings if f.severity == "critical"]
    assert len(critical) == 1
    assert critical[0].rule_id == "SEC-SECRETS-001"
    assert critical[0].file_path == "config/aws.py"
```

---

## TypeScript / React (Dashboard)

### Style & Formatting
- **Formatter:** Prettier
- **Linter:** ESLint with `@typescript-eslint`
- **TypeScript:** strict mode enabled

### Component Style

Use functional components with hooks. No class components:

```tsx
// ✅ Good
interface FindingCardProps {
  finding: Finding;
  onDismiss?: (id: string) => void;
}

export function FindingCard({ finding, onDismiss }: FindingCardProps) {
  return (
    <div className={`finding-card finding-card--${finding.severity}`}>
      <h3>{finding.title}</h3>
      <p>{finding.detail}</p>
    </div>
  );
}

// ❌ Bad: class component
export class FindingCard extends React.Component<FindingCardProps> { ... }
```

### API Calls

Use `react-query` (TanStack Query) for all data fetching. No raw `fetch` calls in components:

```tsx
// ✅ Good
function AnalysisPage({ id }: { id: string }) {
  const { data, isLoading, error } = useQuery({
    queryKey: ['analysis', id],
    queryFn: () => api.getAnalysis(id),
  });
  ...
}
```

---

## SQL (PostgreSQL)

- All queries use parameterized statements — no string formatting
- Migrations use `alembic` (Python) or `flyway` (JVM)
- Every table has `created_at` and `updated_at` columns
- Foreign keys have explicit ON DELETE behavior

```python
# ✅ Good
result = await db.execute(
    select(AnalysisRun).where(AnalysisRun.repo_url == repo_url)
)

# ❌ Bad (SQL injection risk)
result = await db.execute(
    f"SELECT * FROM analysis_runs WHERE repo_url = '{repo_url}'"
)
```

---

## Cypher (Neo4j)

- Always use parameters, never string interpolation
- Include `LIMIT` on all queries that could return large result sets
- Name all relationships descriptively: `[:HAS_FINDING]` not `[:R]`

```cypher
// ✅ Good
MATCH (run:AnalysisRun {id: $run_id})-[:PRODUCED]->(f:Finding)
WHERE f.severity = $severity
RETURN f.title, f.file_path
LIMIT 100

// ❌ Bad
MATCH (run)-[r]->(f) RETURN f
```

---

## Git Conventions

### Branch Naming
```
feature/short-description
fix/short-description
docs/short-description
chore/short-description
```

### Commit Messages (Conventional Commits)
```
feat: add dependency risk skill
fix: handle timeout in skill runner
docs: update api-design.md
chore: upgrade tree-sitter to 0.21
refactor: split graph writer into submodules
test: add integration tests for webhook ingestion
```

### Pull Requests
- Title: same format as commit messages
- Must pass all CI checks before merge
- Requires 1 reviewer approval
- Squash merge to main

---

## Security Coding Rules

1. **Never log secrets** — sanitize before logging
2. **Never store secrets in env vars baked into images** — use Kubernetes Secrets + External Secrets Operator
3. **Always validate external inputs** — Pydantic models at API boundary
4. **Use parameterized queries** — for all database access
5. **Set timeouts on all HTTP calls** — `httpx.AsyncClient(timeout=30)`
6. **Validate webhook signatures** — HMAC-SHA256 always

---

## Related Documents

- [Module Design Guidelines](module-design-guidelines.md)
- [Plugin/Skill Framework](plugin-skill-framework.md)
- [System Design](../docs/system-design.md)
