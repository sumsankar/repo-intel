# High-Level Architecture

## System Overview

The AI Engineering Intelligence Platform is a multi-tier system composed of seven layers, each with a clear responsibility boundary. Data flows in one direction: from raw repositories at the bottom to human-readable reports at the top.

---

## Architecture Diagram

```mermaid
graph TB
    subgraph External["External Systems"]
        GH[GitHub]
        GL[GitLab]
        AZ[Azure DevOps]
        LLM_EXT[LLM API<br/>Anthropic / OpenAI]
        TRIVY[Trivy Vulnerability DB]
        OSV[OSV API]
    end

    subgraph Ingestion["Layer 1 — Ingestion"]
        WH[Webhook Receiver]
        SCHED[Scheduler]
        CL[Repository Cloner]
        META[Metadata Extractor]
    end

    subgraph Parsing["Layer 2 — Parsing"]
        TS[Tree-sitter Parser]
        AST[AST Analyzer]
        IG[Import Graph Builder]
    end

    subgraph Analysis["Layer 3 — Analysis Engines (parallel)"]
        CE[Code Engine]
        AE[Architecture Engine]
        SE[Security Engine]
        DE[DevOps Engine]
        DR[Dependency Engine]
    end

    subgraph Graph["Layer 4 — Knowledge Graph"]
        NEO[Neo4j]
    end

    subgraph AI["Layer 5 — AI Reasoning"]
        ORCH[LLM Orchestrator]
        CACHE[Response Cache]
    end

    subgraph Gov["Layer 6 — Governance"]
        POL[Policy Engine]
        SCORE[Scorer]
    end

    subgraph Output["Layer 7 — Output"]
        RPT[Report Generator]
        API[REST API]
        DASH[React Dashboard]
    end

    GH & GL & AZ --> WH
    SCHED --> CL
    WH --> CL
    CL --> META
    META --> TS
    TS --> AST --> IG

    AST --> CE & AE & SE & DE
    IG --> AE
    DR -.->|calls| TRIVY
    DR -.->|calls| OSV

    CE & AE & SE & DE & DR --> NEO
    NEO --> ORCH
    ORCH -.->|calls| LLM_EXT
    ORCH --> CACHE
    ORCH --> POL
    POL --> SCORE
    SCORE --> RPT
    RPT --> API
    API --> DASH
```

---

## Layer Responsibilities

| Layer | Components | Responsibility |
|-------|-----------|---------------|
| 1. Ingestion | Webhook Receiver, Scheduler, Cloner, Metadata Extractor | Accept repository references, clone to local disk, extract metadata |
| 2. Parsing | Tree-sitter Parser, AST Analyzer, Import Graph Builder | Generate language-agnostic structural representation |
| 3. Analysis | 5 parallel engines | Evaluate repository across all quality dimensions |
| 4. Knowledge Graph | Neo4j | Persist findings in queryable, persistent graph |
| 5. AI Reasoning | LLM Orchestrator, Cache | Synthesize findings into natural language insights |
| 6. Governance | Policy Engine, Scorer | Evaluate compliance against organizational standards |
| 7. Output | Report Generator, REST API, Dashboard | Deliver findings to users in usable formats |

---

## Key Design Decisions

### Decision 1: AI Augments Static Analysis (Not Replaces It)

Static analysis runs first and produces structured `Finding` objects with specific file and line references. The AI layer only receives these structured findings — it never reads raw source code. This means:
- AI output is grounded in verifiable facts
- No risk of the AI "inventing" findings
- Token costs are controlled (findings are much smaller than source code)
- The platform still works if the AI service is unavailable

### Decision 2: Skill-Based Extensibility

Every analysis dimension is a self-contained `Skill` module. Adding a new analysis type requires only:
1. Creating a new skill file (`skills/your-skill.md` or Python class)
2. Registering it in the skill runner

No changes to core infrastructure. This follows the Open/Closed Principle.

### Decision 3: Graph-First Knowledge Storage

All findings are stored as nodes/edges in Neo4j rather than flat JSON documents. This enables:
- Questions that span multiple dimensions: "show files with both secrets and no tests"
- Portfolio-level queries across repositories
- Temporal queries: "what changed between last week and this week?"
- Relationship traversal: "which repos use a vulnerable version of log4j?"

### Decision 4: Ephemeral Clones

Repository clones exist only for the duration of analysis. After Stage 7, the clone is deleted. This means:
- No persistent storage of customer code
- Disk usage is bounded by concurrent analyses
- Security: cloned code never persists in the system

### Decision 5: Fail-Safe Analysis

Each skill runs in isolation. If the DevOps skill times out, the Security, Code, and Architecture skills still complete and their findings are reported. The report notes which skills completed and which were skipped.

---

## Component Interaction Model

```mermaid
sequenceDiagram
    actor Dev as Developer / CI
    participant API as API Server
    participant Q as Job Queue
    participant W as Analysis Worker
    participant NEO as Neo4j
    participant AI as LLM API

    Dev->>API: POST /analyses {repo_url}
    API->>Q: Enqueue job
    API-->>Dev: 202 {analysis_id}

    Q->>W: Dequeue job
    W->>W: Clone + Parse
    par Skills run in parallel
        W->>W: Code Skill
        W->>W: Security Skill
        W->>W: Architecture Skill
        W->>W: DevOps Skill
        W->>W: Dependency Skill
    end
    W->>NEO: Write findings graph
    W->>AI: Synthesize insights
    AI-->>W: Summaries + roadmap
    W->>W: Generate report
    W->>API: Store report

    Dev->>API: GET /analyses/{id}/report.md
    API-->>Dev: Report markdown
```

---

## Failure Domains

The architecture is designed so that failure in one domain does not cascade:

| Component fails | Impact | Mitigation |
|----------------|--------|-----------|
| LLM API | No AI summaries in report | Return raw findings without summaries |
| Neo4j | No trend data, no graph queries | Analysis still completes, results stored in PostgreSQL |
| Trivy/OSV | No CVE data | Dependency skill reports "scan unavailable" |
| Single worker | Queued jobs processed by other workers | HPA adds workers if queue depth grows |
| Redis queue | No new analyses triggered | API returns 503; existing analyses continue |

---

## Related Documents

- [Component Architecture](component-architecture.md)
- [Microservices Architecture](microservices-architecture.md)
- [Data Flow](data-flow.md)
- [System Design](../docs/system-design.md)
