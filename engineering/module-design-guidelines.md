# Module Design Guidelines

## Core Principles

Every module in this platform is designed around three rules:

1. **Single Responsibility** — one module does one thing well
2. **Explicit Dependencies** — no hidden globals; dependencies are injected
3. **Testable in Isolation** — modules can be unit-tested without starting a database or external service

---

## Module Structure

### Backend Service Layout

```
service-name/
├── src/
│   ├── api/
│   │   ├── routes/
│   │   │   ├── analyses.py       # Route handlers only
│   │   │   ├── findings.py
│   │   │   └── portfolio.py
│   │   ├── middleware/
│   │   │   ├── auth.py
│   │   │   └── request_id.py
│   │   └── main.py               # FastAPI app factory
│   ├── domain/
│   │   ├── models.py             # Dataclasses (no ORM)
│   │   ├── skill.py              # Skill protocol
│   │   └── finding.py
│   ├── skills/
│   │   ├── base.py               # BaseSkill
│   │   ├── code_skill.py
│   │   ├── security_skill.py
│   │   ├── architecture_skill.py
│   │   ├── devops_skill.py
│   │   └── dependency_skill.py
│   ├── services/
│   │   ├── analysis_service.py   # Orchestrates worker pipeline
│   │   ├── graph_service.py      # Neo4j client wrapper
│   │   ├── ai_service.py         # LLM orchestration
│   │   └── report_service.py
│   ├── repositories/
│   │   ├── analysis_repo.py      # PostgreSQL access layer
│   │   └── job_repo.py
│   ├── infrastructure/
│   │   ├── neo4j.py              # Neo4j driver factory
│   │   ├── redis.py              # Redis client factory
│   │   ├── postgres.py           # SQLAlchemy session factory
│   │   └── llm.py                # LLM provider factory
│   └── config.py                 # Settings (pydantic-settings)
├── tests/
│   ├── unit/                     # No external services
│   │   ├── skills/
│   │   │   ├── test_code_skill.py
│   │   │   └── test_security_skill.py
│   │   └── services/
│   ├── integration/              # Real database, fake LLM
│   │   ├── test_analysis_flow.py
│   │   └── test_graph_writer.py
│   └── e2e/                      # Full stack, real services
│       └── test_webhook_to_report.py
├── Dockerfile
├── pyproject.toml
└── README.md
```

---

## Dependency Injection Pattern

Use FastAPI's `Depends()` system for dependency injection. Never access services as module-level globals:

```python
# ✅ Good: dependencies injected
class AnalysisService:
    def __init__(
        self,
        job_repo: JobRepository,
        graph_client: GraphServiceClient,
        ai_client: AIServiceClient,
    ):
        self.job_repo = job_repo
        self.graph_client = graph_client
        self.ai_client = ai_client

# FastAPI route
async def create_analysis(
    body: CreateAnalysisRequest,
    service: AnalysisService = Depends(get_analysis_service),
):
    return await service.create(body.repo_url, body.skills)

# ❌ Bad: global singleton
_neo4j_driver = None  # global state

async def write_to_graph(findings):
    global _neo4j_driver  # hidden dependency
    ...
```

---

## Layering Rules

The codebase is strictly layered. Each layer may only call the layer directly below it:

```
API Routes
    ↓ (calls)
Services
    ↓ (calls)
Repositories / External Clients
    ↓ (calls)
Infrastructure (DB connections, HTTP clients)
```

**Violations are caught by architecture analysis:**
- Routes may NOT call repositories directly
- Services may NOT import from other services (use events or explicit orchestration)
- Infrastructure modules may NOT import from domain or services

---

## Skill Module Pattern

Every skill follows the same structure:

```python
# skills/security_skill.py

from src.domain.skill import Skill, SkillResult
from src.domain.models import RepoContext, Finding

class SecuritySkill(Skill):
    name = "security"
    weight = 0.30

    def __init__(self, semgrep_runner: SemgrepRunner):
        self.semgrep = semgrep_runner

    async def analyze(self, context: RepoContext) -> SkillResult:
        start = time.monotonic()

        findings: list[Finding] = []
        findings.extend(await self._scan_secrets(context))
        findings.extend(await self._run_semgrep(context))
        findings.extend(await self._check_sensitive_files(context))
        findings.extend(await self._check_gitignore(context))

        return SkillResult(
            skill_name=self.name,
            status="complete",
            findings=findings,
            metrics=self._compute_metrics(findings),
            score=self.score(findings),
            elapsed_ms=round((time.monotonic() - start) * 1000)
        )

    async def _scan_secrets(self, context: RepoContext) -> list[Finding]:
        # Implementation
        ...

    async def _run_semgrep(self, context: RepoContext) -> list[Finding]:
        # Implementation
        ...
```

**Rules for skills:**
- Skills must not have side effects (no writing to DB, no HTTP calls to our own services)
- Skills must not depend on each other
- Skills must be independently testable with a mock `RepoContext`
- Skills must handle their own errors and return partial results rather than raising

---

## Configuration Management

All configuration comes from environment variables via `pydantic-settings`:

```python
# src/config.py
from pydantic_settings import BaseSettings, SettingsConfigDict

class Settings(BaseSettings):
    model_config = SettingsConfigDict(env_file=".env", case_sensitive=False)

    # Required
    database_url: PostgresDsn
    redis_url: str
    neo4j_uri: str
    neo4j_auth: str

    # Optional with defaults
    anthropic_api_key: str | None = None
    openai_api_key: str | None = None
    llm_provider: Literal["anthropic", "openai"] = "anthropic"
    llm_model: str = "claude-sonnet-4-6"

    skill_timeout_seconds: int = 120
    max_repo_size_mb: int = 2048

    log_level: str = "INFO"
    environment: Literal["development", "staging", "production"] = "development"

# Singleton pattern — created once at startup
_settings: Settings | None = None

def get_settings() -> Settings:
    global _settings
    if _settings is None:
        _settings = Settings()
    return _settings
```

---

## Interface-Based Design

Define interfaces for external dependencies so they can be swapped in tests:

```python
# domain/interfaces.py

class LLMProvider(Protocol):
    async def complete(
        self,
        system: str,
        user: str,
        max_tokens: int = 2048,
    ) -> str: ...

class GraphClient(Protocol):
    async def write_run(self, run: AnalysisRun) -> None: ...
    async def query(self, cypher: str, params: dict) -> list[dict]: ...

# In tests:
class MockLLMProvider:
    def __init__(self, response: str):
        self._response = response

    async def complete(self, system, user, max_tokens=2048) -> str:
        return self._response
```

---

## File Size Limits

| File Type | Max Lines | Action if exceeded |
|-----------|-----------|-------------------|
| Route handler | 100 | Split into sub-routers |
| Service class | 300 | Split into focused sub-services |
| Skill class | 400 | Extract helpers into `_helpers.py` |
| Test file | 500 | Split by function group |
| Config | 100 | Split into sub-configs |

---

## Related Documents

- [Coding Standards](coding-standards.md)
- [Plugin/Skill Framework](plugin-skill-framework.md)
- [Component Architecture](../architecture/component-architecture.md)
