# Finding Schema & Rule Registry

**Normative.** Every finding emitted by any skill MUST be expressible as one of the rule IDs below. New checks require a new rule ID, added here, before they ship.

---

## 1. Rule ID format

```
RI-<SKILL>-<NNN>-<SLUG>
```

- `SKILL`  — three-letter code: `SEC`, `CODE`, `ARCH`, `DEVOPS`, `DEP`, `GOV`.
- `NNN`    — zero-padded three-digit sequence, unique within the skill.
- `SLUG`   — UPPERCASE-WITH-HYPHENS short description, no spaces.

Rule IDs are immutable once published. To change the meaning, deprecate the old ID and register a new one.

---

## 2. Finding fields (wire format)

Every finding MUST include:

| Field | Type | Required | Description |
|-------|------|:--------:|-------------|
| `ruleId`          | string (`RI-*`)           | ✅ | Must exist in this registry. |
| `severity`        | enum                      | ✅ | `critical \| high \| medium \| low \| info`. |
| `file`            | string                    | ✅ | Repo-relative path with forward slashes. |
| `line`            | integer                   |    | 1-indexed. Omit if not line-specific. |
| `endLine`, `endColumn`, `startColumn` | integer | | 1-indexed, optional precision. |
| `message`         | string                    | ✅ | One sentence describing the finding. |
| `fix`             | string                    | ✅ | Actionable remediation, one sentence. |
| `cwe`             | string (`CWE-###`)        |    | Set if the rule registry declares a CWE mapping. |
| `owasp`           | string (e.g. `A01:2021`)  |    | Same. |
| `package`         | string                    |    | Monorepo package name if applicable. |
| `evidence`        | string                    |    | Short excerpt (≤ 120 chars). Must be HTML-escaped before template splicing. |
| `partialFingerprint` | string (hex)            |    | Hash that survives unrelated edits; used for dedup across runs. |

Additional fields are allowed if prefixed with `properties.*` and will be passed through to SARIF.

---

## 3. Canonical rule registry (v1.0)

Each row lists: `rule_id`, default severity, CWE/OWASP mapping, and `base_impact` (see SCORING-CONTRACT.md).

### Security (`RI-SEC-*`)

| ID | Name | Default severity | CWE | OWASP | Base impact |
|----|------|------------------|-----|-------|------------:|
| RI-SEC-001-HARDCODED-SECRET | Hardcoded secret in source | critical | CWE-798 | A07:2021 | 0.7 |
| RI-SEC-002-SQL-INJECTION-RISK | Unsanitised SQL concatenation | high | CWE-89 | A03:2021 | 0.6 |
| RI-SEC-003-COMMAND-INJECTION-RISK | Unsanitised shell invocation | high | CWE-78 | A03:2021 | 0.6 |
| RI-SEC-004-XSS-RISK | Unsanitised HTML interpolation | high | CWE-79 | A03:2021 | 0.5 |
| RI-SEC-005-WEAK-CRYPTO | Use of MD5/SHA1 for security | medium | CWE-327 | A02:2021 | 0.4 |
| RI-SEC-006-INSECURE-DESERIALIZATION | `pickle`/`yaml.load`/etc. on untrusted input | high | CWE-502 | A08:2021 | 0.6 |
| RI-SEC-007-MISSING-AUTHZ-CHECK | Endpoint with no auth guard | high | CWE-862 | A01:2021 | 0.5 |
| RI-SEC-008-PERMISSIVE-CORS | `Access-Control-Allow-Origin: *` with credentials | medium | CWE-942 | A05:2021 | 0.4 |
| RI-SEC-009-MISSING-GITIGNORE-SECRETS | `.env` not in `.gitignore` | medium | CWE-540 | A05:2021 | 0.3 |
| RI-SEC-010-OUTDATED-TLS | TLS 1.0/1.1 configured | medium | CWE-326 | A02:2021 | 0.3 |
| RI-SEC-011-SWALLOWED-EXCEPTION | Empty `catch` with no logging | low | CWE-391 | — | 0.2 |

### Code quality (`RI-CODE-*`)

| ID | Name | Default severity | Base impact |
|----|------|------------------|------------:|
| RI-CODE-001-LARGE-FILE | File exceeds large-file threshold | medium | 0.3 |
| RI-CODE-002-GOD-OBJECT | File exceeds god-object threshold | high | 0.5 |
| RI-CODE-003-NO-TESTS | Zero test files detected | critical | 1.0 |
| RI-CODE-004-LOW-TEST-RATIO | Test ratio below 10% | high | 0.6 |
| RI-CODE-005-DUPLICATION | Duplicated block across files | medium | 0.3 |
| RI-CODE-006-LONG-FUNCTION | Function exceeds 80 lines | medium | 0.3 |
| RI-CODE-007-MISSING-README | README absent or empty | medium | 0.3 |
| RI-CODE-008-HIGH-TODO-DENSITY | >1 TODO/FIXME per 100 LOC | low | 0.2 |
| RI-CODE-009-DEEP-NESTING | Block nesting > 5 | low | 0.2 |

### Architecture (`RI-ARCH-*`)

| ID | Name | Default severity | Base impact |
|----|------|------------------|------------:|
| RI-ARCH-001-CIRCULAR-DEPENDENCY | Mutual import cycle between modules | high | 0.7 |
| RI-ARCH-002-NO-LAYERING | Flat `src/` with no separation of concerns | medium | 0.5 |
| RI-ARCH-003-LAYER-VIOLATION | Lower layer importing higher layer | high | 0.5 |
| RI-ARCH-004-TIGHT-COUPLING | Module imports >15 distinct modules | medium | 0.3 |
| RI-ARCH-005-MISSING-ABSTRACTION | Direct external-dependency usage spread across code | low | 0.3 |
| RI-ARCH-006-INCONSISTENT-PATTERNS | Same concern implemented >2 different ways | medium | 0.3 |
| RI-ARCH-007-WORKSPACE-DRIFT | Monorepo packages at inconsistent versions of a shared dep | medium | 0.3 |

### DevOps (`RI-DEVOPS-*`)

| ID | Name | Default severity | Base impact |
|----|------|------------------|------------:|
| RI-DEVOPS-001-NO-CI | No CI workflow detected | critical | 1.0 |
| RI-DEVOPS-002-OUTDATED-ACTIONS | `actions/*@v1` or `@v2` references | high | 0.4 |
| RI-DEVOPS-003-NO-SECURITY-SCAN | No SAST / secret scan / SCA step in CI | high | 0.6 |
| RI-DEVOPS-004-UNPINNED-DEPS-IN-DOCKER | `apt-get install` without versions | medium | 0.3 |
| RI-DEVOPS-005-ROOT-CONTAINER | Container runs as root | high | 0.4 |
| RI-DEVOPS-006-MISSING-HEALTHCHECK | No `HEALTHCHECK` in Dockerfile / missing liveness probe | medium | 0.3 |
| RI-DEVOPS-007-PLAINTEXT-SECRETS-IN-IAC | Secret literal in Terraform/Helm/K8s manifest | critical | 0.8 |
| RI-DEVOPS-008-NO-LOCK-FILE | Dependency lock file missing | medium | 0.4 |
| RI-DEVOPS-009-NO-DOCKERIGNORE | Dockerfile without .dockerignore | low | 0.2 |

### Dependency (`RI-DEP-*`)

| ID | Name | Default severity | Base impact |
|----|------|------------------|------------:|
| RI-DEP-001-KNOWN-CVE-CRITICAL | Dependency with a critical CVE | critical | 1.0 |
| RI-DEP-002-KNOWN-CVE-HIGH | Dependency with a high CVE | high | 0.7 |
| RI-DEP-003-KNOWN-CVE-MEDIUM | Dependency with a medium CVE | medium | 0.4 |
| RI-DEP-004-DEPRECATED-PACKAGE | Deprecated / unmaintained package | medium | 0.4 |
| RI-DEP-005-LICENSE-RISK | Copyleft / non-commercial licence in prod deps | high | 0.5 |
| RI-DEP-006-UNLICENSED | Dependency with no declared licence | medium | 0.3 |
| RI-DEP-007-OUTDATED-MAJOR | >2 major versions behind latest | low | 0.2 |
| RI-DEP-008-DIRECT-TRANSITIVE-CONFLICT | Lockfile has conflicting versions of same package | medium | 0.3 |

### Governance (`RI-GOV-*`)

Governance rules wrap other skills' rules and map to organisation-level policy IDs. Base impacts are lower because the underlying skill already scored the issue; governance tracks *compliance posture*, not raw severity.

| ID | Policy | Default severity | Base impact |
|----|--------|------------------|------------:|
| RI-GOV-001-POLICY-FAILURE-SECURITY | Any critical `RI-SEC-*` finding | critical | 0.5 |
| RI-GOV-002-POLICY-FAILURE-TESTS | Test ratio below 10% (maps to GOV-CODE-001) | high | 0.4 |
| RI-GOV-003-POLICY-FAILURE-CI | No CI (maps to GOV-DEVOPS-001) | high | 0.4 |
| RI-GOV-004-POLICY-FAILURE-LICENSE | Copyleft in prod (maps to GOV-DEP-002) | high | 0.3 |
| RI-GOV-005-POLICY-FAILURE-DOCS | README missing (maps to GOV-CODE-002) | medium | 0.2 |

---

## 4. Category → OWASP/CWE mapping

`properties.category` on each finding is a free-form short label (e.g., `secrets`, `injection`). The **authoritative** taxonomy links are `cwe` and `owasp` on the rule definition, not the category string. Category is for display only; automated consumers should use `ruleId` or `cwe`.

---

## 5. Adding a new rule

1. Pick the next `NNN` in the skill namespace (do not reuse).
2. Add a row to the table above with severity, CWE/OWASP, and `base_impact`.
3. Add a `## Rules` section to the skill file declaring how the rule fires (the detection pattern).
4. Add a CHANGELOG entry. Rule additions are minor version bumps.

## 6. Deprecating a rule

Mark the row with `~~strikethrough~~` and add `(deprecated since vX.Y, replaced by RI-...)`. Deprecated rules keep firing for one major version to preserve fingerprints and dedup; remove them only on the next major bump.
