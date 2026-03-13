# API Design

## Overview

The platform exposes a RESTful HTTP API for triggering analyses, retrieving reports, querying the knowledge graph, and managing governance policies. The API follows REST conventions and returns JSON responses.

**Base URL:** `https://api.repo-intel.example.com/api/v1`

**Authentication:** Bearer token (JWT) in `Authorization` header

---

## Authentication

```bash
# Obtain a token
POST /api/v1/auth/token
Content-Type: application/json

{
  "client_id": "your-client-id",
  "client_secret": "your-client-secret"
}

# Response
{
  "access_token": "eyJhbGc...",
  "token_type": "bearer",
  "expires_in": 3600
}

# Use token in requests
Authorization: Bearer eyJhbGc...
```

---

## Analysis Endpoints

### Trigger Analysis

```http
POST /api/v1/analyses
Content-Type: application/json
Authorization: Bearer {token}

{
  "repo_url": "https://github.com/org/repo",
  "skills": ["all"],               // or ["security", "devops"]
  "branch": "main",                // optional, defaults to default branch
  "callback_url": "https://...",   // optional webhook on completion
  "governance_policy": "strict"    // optional, named policy from config
}
```

**Response `202 Accepted`:**
```json
{
  "analysis_id": "ana_01HXYZ123",
  "status": "queued",
  "repo_url": "https://github.com/org/repo",
  "estimated_duration_seconds": 90,
  "poll_url": "/api/v1/analyses/ana_01HXYZ123",
  "created_at": "2026-03-12T10:00:00Z"
}
```

---

### Get Analysis Status

```http
GET /api/v1/analyses/{analysis_id}
Authorization: Bearer {token}
```

**Response `200 OK` (while running):**
```json
{
  "analysis_id": "ana_01HXYZ123",
  "status": "running",
  "stage": "security_analysis",
  "progress_percent": 45,
  "started_at": "2026-03-12T10:00:05Z"
}
```

**Response `200 OK` (completed):**
```json
{
  "analysis_id": "ana_01HXYZ123",
  "status": "complete",
  "repo_url": "https://github.com/org/repo",
  "overall_score": 6.2,
  "risk_level": "medium",
  "is_compliant": true,
  "scores": {
    "code": 7,
    "architecture": 6,
    "security": 6,
    "devops": 5,
    "dependency": 8
  },
  "critical_count": 0,
  "high_count": 3,
  "medium_count": 8,
  "low_count": 12,
  "report_urls": {
    "markdown": "/api/v1/analyses/ana_01HXYZ123/report.md",
    "html":     "/api/v1/analyses/ana_01HXYZ123/report.html",
    "json":     "/api/v1/analyses/ana_01HXYZ123/report.json"
  },
  "started_at": "2026-03-12T10:00:05Z",
  "completed_at": "2026-03-12T10:01:38Z",
  "duration_seconds": 93
}
```

---

### Get Report

```http
GET /api/v1/analyses/{analysis_id}/report.md
Authorization: Bearer {token}
Accept: text/markdown

# Returns: full Markdown report
```

```http
GET /api/v1/analyses/{analysis_id}/report.html
Authorization: Bearer {token}
Accept: text/html

# Returns: self-contained HTML report
```

```http
GET /api/v1/analyses/{analysis_id}/report.json
Authorization: Bearer {token}
Accept: application/json

# Returns: machine-readable structured findings
```

---

### List Analyses for a Repository

```http
GET /api/v1/repositories/{repo_encoded_url}/analyses
Authorization: Bearer {token}

# repo_encoded_url = URL-encoded repo URL
# e.g. github.com%2Forg%2Frepo

?limit=10&offset=0&order=desc
```

**Response `200 OK`:**
```json
{
  "items": [
    {
      "analysis_id": "ana_01HXYZ123",
      "status": "complete",
      "overall_score": 6.2,
      "completed_at": "2026-03-12T10:01:38Z"
    }
  ],
  "total": 42,
  "limit": 10,
  "offset": 0
}
```

---

## Findings Endpoints

### List Findings

```http
GET /api/v1/analyses/{analysis_id}/findings
Authorization: Bearer {token}

?severity=critical,high     // filter by severity
&category=secrets,injection // filter by category
&skill=security             // filter by skill
&limit=50&offset=0
```

**Response `200 OK`:**
```json
{
  "items": [
    {
      "id": "fnd_abc123",
      "severity": "critical",
      "category": "secrets",
      "skill": "security",
      "title": "AWS Access Key committed to repository",
      "detail": "AWS Access Key ID AKIA... found in config/aws.js line 12",
      "file_path": "config/aws.js",
      "line_number": 12,
      "fix": "Rotate this key immediately. Remove from code and use environment variables.",
      "rule_id": "SEC-SECRETS-001"
    }
  ],
  "total": 3,
  "counts": {
    "critical": 1,
    "high": 2,
    "medium": 8,
    "low": 14
  }
}
```

---

## Knowledge Graph Query Endpoint

```http
POST /api/v1/graph/query
Content-Type: application/json
Authorization: Bearer {token}

{
  "cypher": "MATCH (f:Finding {severity: 'critical'})-[:LOCATED_IN]->(file:File) RETURN f.title, file.path LIMIT 10",
  "repo_url": "https://github.com/org/repo"  // scope query to one repo
}
```

**Response `200 OK`:**
```json
{
  "columns": ["f.title", "file.path"],
  "rows": [
    ["AWS Access Key committed", "config/aws.js"],
    ["Firebase private key committed", "service-account.json"]
  ],
  "duration_ms": 14
}
```

Note: Only read-only Cypher queries are permitted. Mutations are rejected.

---

## Portfolio Endpoints

### Portfolio Score Summary

```http
GET /api/v1/portfolio/scores
Authorization: Bearer {token}

?org=my-github-org    // optional: filter by org
&days=30              // look back window for "latest" run
```

**Response `200 OK`:**
```json
{
  "summary": {
    "total_repositories": 15,
    "compliant": 9,
    "non_compliant": 3,
    "not_analyzed": 3,
    "average_score": 6.4,
    "critical_repositories": 2
  },
  "repositories": [
    {
      "repo_url": "https://github.com/org/payment-service",
      "overall_score": 8.2,
      "risk_level": "low",
      "is_compliant": true,
      "last_analyzed": "2026-03-12T09:00:00Z"
    }
  ]
}
```

---

## Governance Endpoints

### Evaluate Governance Compliance

```http
POST /api/v1/governance/evaluate
Content-Type: application/json
Authorization: Bearer {token}

{
  "analysis_id": "ana_01HXYZ123",
  "policy": "strict"   // named policy, or inline policy object
}
```

**Response `200 OK`:**
```json
{
  "is_compliant": false,
  "governance_score": 4.3,
  "risk_posture_index": 31,
  "blocking_violations": [
    {
      "policy_id": "SEC-001",
      "name": "No committed secrets",
      "severity": "critical",
      "recommendation": "Revoke credentials at config/aws.js:12"
    }
  ],
  "advisory_violations": [
    {
      "policy_id": "CODE-001",
      "name": "Minimum test coverage",
      "severity": "high",
      "recommendation": "Current test ratio 3%, target 10%"
    }
  ],
  "passed_policies": ["DEP-002", "ARCH-001"]
}
```

---

## Webhook Management

### Register Webhook

```http
POST /api/v1/webhooks
Content-Type: application/json
Authorization: Bearer {token}

{
  "provider": "github",
  "repo_url": "https://github.com/org/repo",
  "secret": "your-webhook-secret",
  "events": ["push", "pull_request"]
}
```

---

## Error Responses

All errors follow a consistent format:

```json
{
  "error": {
    "code": "REPO_NOT_FOUND",
    "message": "Repository not found or access denied",
    "detail": "Ensure the repository URL is correct and your access token has read permissions",
    "request_id": "req_abc123"
  }
}
```

| HTTP Status | Error Code | Meaning |
|-------------|-----------|---------|
| 400 | `INVALID_REQUEST` | Malformed request body |
| 401 | `UNAUTHORIZED` | Missing or invalid token |
| 403 | `FORBIDDEN` | Token lacks required permissions |
| 404 | `NOT_FOUND` | Analysis or repository not found |
| 409 | `ALREADY_RUNNING` | Analysis already in progress for this repo |
| 429 | `RATE_LIMITED` | Too many requests |
| 503 | `SERVICE_UNAVAILABLE` | Worker pool at capacity |

---

## Rate Limits

| Tier | Analyses/hour | API calls/minute |
|------|--------------|-----------------|
| Free | 5 | 60 |
| Team | 50 | 300 |
| Enterprise | Unlimited | 1,000 |

Rate limit headers:
```
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
X-RateLimit-Reset: 1710240000
```

---

## SDK Examples

### Python SDK

```python
from repo_intel import Client

client = Client(api_key="your-api-key")

# Trigger analysis
analysis = client.analyses.create(
    repo_url="https://github.com/org/repo",
    skills=["all"]
)

# Poll until complete
result = analysis.wait(timeout=300)

print(f"Score: {result.overall_score}/10")
print(f"Critical findings: {result.critical_count}")
print(f"Report: {result.report_urls['markdown']}")
```

### GitHub Actions Integration

```yaml
- name: Analyze repository
  uses: repo-intel/analyze-action@v1
  with:
    api_key: ${{ secrets.REPO_INTEL_API_KEY }}
    fail_on_critical: true
    fail_if_non_compliant: true
    skills: security,devops,dependency
```

---

## Related Documents

- [System Design](system-design.md)
- [Deployment Architecture](deployment-architecture.md)
- [Knowledge Graph](knowledge-graph.md)
