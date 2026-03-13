# System Architecture

## Architectural Principles

The platform is designed around five core principles:

1. **Skill-based extensibility** — Analysis capabilities are modular skills that can be added without modifying the core engine
2. **Graph-first knowledge** — All relationships are stored in a knowledge graph, enabling cross-cutting queries
3. **AI-augmented, not AI-dependent** — Static analysis runs first; AI adds synthesis and natural language reasoning on top
4. **Fail-safe analysis** — Each skill runs independently; one failure does not block others
5. **Immutable reports** — Generated reports are versioned snapshots, not live views

---

## Layered Architecture

```mermaid
graph TB
    subgraph L1["Layer 1 — Ingestion"]
        WH[Webhook Receiver]
        CL[Repository Cloner]
        ME[Metadata Extractor]
    end

    subgraph L2["Layer 2 — Parsing"]
        TS[Tree-sitter Parser]
        AST[AST Generator]
        DG[Dependency Graph Builder]
    end

    subgraph L3["Layer 3 — Analysis Engines"]
        CE[Code Quality Engine]
        AE[Architecture Engine]
        SE[Security Engine]
        DE[DevOps Engine]
        DR[Dependency Risk Engine]
    end

    subgraph L4["Layer 4 — Knowledge Graph"]
        NEO[Neo4j Graph DB]
        IDX[Search Index]
    end

    subgraph L5["Layer 5 — AI Reasoning"]
        LLM[LLM Orchestrator]
        PM[Prompt Manager]
        IS[Insight Synthesizer]
    end

    subgraph L6["Layer 6 — Governance"]
        GS[Governance Scorer]
        PR[Policy Ruleset Engine]
        RS[Risk Scorer]
    end

    subgraph L7["Layer 7 — Output"]
        RG[Report Generator]
        API[REST API]
        UI[React Dashboard]
    end

    L1 --> L2 --> L3 --> L4 --> L5 --> L6 --> L7
```

---

## Component Overview

### Ingestion Layer

| Component | Responsibility | Technology |
|-----------|---------------|------------|
| Webhook Receiver | Receives push/PR events from GitHub/GitLab | FastAPI endpoint |
| Repository Cloner | Shallow-clones repositories to local disk | `gitpython`, `subprocess` |
| Metadata Extractor | Extracts language breakdown, file counts, commit stats | `pygments`, `gitpython` |

### Parsing Layer

| Component | Responsibility | Technology |
|-----------|---------------|------------|
| Tree-sitter Parser | Generates ASTs for 40+ languages | `tree-sitter` |
| AST Generator | Extracts functions, classes, imports, exports | Custom tree-sitter grammars |
| Dependency Graph Builder | Maps import/require/using relationships | AST traversal |

### Analysis Engines

| Engine | What it analyzes | Tools |
|--------|-----------------|-------|
| Code Quality Engine | Complexity, test coverage, duplication | Custom metrics + AST |
| Architecture Engine | Structure, patterns, coupling | Dependency graph traversal |
| Security Engine | Secrets, vulnerabilities, insecure patterns | Semgrep, custom regex |
| DevOps Engine | CI/CD, Docker, IaC, hygiene | File pattern matching |
| Dependency Risk Engine | CVEs, license risk, supply chain | Trivy, OSV API |

### Knowledge Graph Layer

All analysis outputs are written as nodes and edges into a Neo4j graph. This enables:
- Cross-cutting queries (e.g. "find all files that have both security issues and no tests")
- Relationship traversal (e.g. "which services depend on this vulnerable library?")
- Trend analysis across analysis runs

### AI Reasoning Layer

The LLM orchestrator takes the structured knowledge graph output and:
1. Generates natural language summaries per skill dimension
2. Identifies patterns that span multiple dimensions
3. Prioritizes findings by impact
4. Produces executive-level insights

### Governance Layer

Scores each repository against a configurable policy ruleset:
- Architecture compliance (naming conventions, layer boundaries, dependency direction)
- Security posture (minimum score thresholds, mandatory checks)
- Code health (coverage thresholds, complexity limits)
- DevOps maturity (required pipeline stages, hygiene requirements)

---

## Data Flow

```mermaid
sequenceDiagram
    participant GH as GitHub
    participant API as API Server
    participant CL as Cloner
    participant ENG as Analysis Engines
    participant KG as Knowledge Graph
    participant AI as AI Engine
    participant DB as Report Store

    GH->>API: Push webhook event
    API->>CL: Clone repository
    CL->>ENG: Parsed file tree
    ENG->>ENG: Run all skills in parallel
    ENG->>KG: Write findings as graph nodes/edges
    KG->>AI: Query knowledge graph
    AI->>AI: Synthesize cross-cutting insights
    AI->>DB: Store generated report
    DB->>API: Return report via REST
```

---

## Deployment Topology

```mermaid
graph LR
    subgraph Cloud
        LB[Load Balancer]
        subgraph App Tier
            API1[API Server 1]
            API2[API Server 2]
        end
        subgraph Worker Tier
            W1[Analysis Worker 1]
            W2[Analysis Worker 2]
            W3[Analysis Worker 3]
        end
        subgraph Data Tier
            NEO[Neo4j Cluster]
            PG[PostgreSQL]
            RD[Redis Queue]
            S3[Object Storage]
        end
        subgraph AI Tier
            LLM[LLM API Gateway]
        end
    end

    LB --> API1
    LB --> API2
    API1 --> RD
    API2 --> RD
    RD --> W1
    RD --> W2
    RD --> W3
    W1 --> NEO
    W2 --> NEO
    W3 --> NEO
    W1 --> S3
    W1 --> LLM
    API1 --> PG
    API1 --> NEO
```

---

## Technology Decisions

### Why tree-sitter for parsing?
Tree-sitter provides concrete syntax trees (not just tokens) for 40+ languages with a single consistent API. Error recovery means malformed code still produces a partial AST. It's significantly faster than language-specific tools.

### Why Neo4j for the knowledge graph?
The relationships between files, functions, dependencies, and findings are naturally graph-shaped. Neo4j's Cypher query language makes cross-cutting queries (e.g. "find all high-severity findings in files with no tests") simple to express.

### Why a message queue for analysis?
Repository analysis can take 30 seconds to 5 minutes depending on codebase size. A message queue (Redis/RabbitMQ) decouples ingestion from analysis, enables parallel workers, and provides retry/dead-letter semantics.

### Why separate AI and static analysis?
Static analysis is deterministic, fast, and cheap. AI reasoning is probabilistic, slower, and costs money per call. Running static analysis first and feeding structured results to the AI dramatically reduces token costs and improves accuracy.

---

## Security Architecture

- All repository clones are performed in isolated ephemeral containers
- Cloned code never touches the API tier — only analysis results do
- Secrets found during analysis are hashed before storage — never stored in plaintext
- All API endpoints require authentication (JWT)
- Knowledge graph access is tenant-isolated

---

## Related Documents

- [Component Architecture](../architecture/component-architecture.md)
- [Microservices Architecture](../architecture/microservices-architecture.md)
- [Data Flow](../architecture/data-flow.md)
- [Deployment Architecture](deployment-architecture.md)
