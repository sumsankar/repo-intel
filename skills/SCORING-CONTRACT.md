# Scoring Contract

**Normative.** Every scoring skill (`security`, `code`, `architecture`, `devops`, `dependency`, `governance`) MUST follow this contract. Reports that do not conform are invalid.

---

## 1. Scoring model

All skills use the same model:

- Start at **10.0**.
- Apply one deduction per finding, computed as `base_impact × severity_multiplier`.
- Floor at **0.0**.
- Round to one decimal place.

### Severity multipliers

| Severity | Multiplier | SARIF `level` | CVSS band |
|----------|-----------:|---------------|-----------|
| critical | ×3.0 | `error` | 9.0–10.0 |
| high     | ×2.0 | `error` | 7.0–8.9  |
| medium   | ×1.0 | `warning` | 4.0–6.9 |
| low      | ×0.5 | `note` | 0.1–3.9  |
| info     | ×0.0 | `none` | 0.0      |

Example: a `RI-SEC-001-HARDCODED-SECRET` rule with `base_impact: 0.7` fires at `critical` → deducts `0.7 × 3.0 = 2.1`.

### Declaration

Each rule in `skills/FINDING-SCHEMA.md` declares its `base_impact` (a float between 0.1 and 1.0). That number is fixed for the rule. Per-run severity can be adjusted via `repo-intel.yml` `rules.upgrade` / `rules.downgrade`; `base_impact` cannot.

### Caps

- No single rule may deduct more than **3.0** from a dimension, regardless of how many times it fires. Duplicate findings beyond the cap still appear in the report but do not compound the score.
- No dimension score may go below **0.0**.

---

## 2. Dimension weights

The overall score is computed from the six dimension scores:

```
overall =
    security     × 0.30
  + code         × 0.20
  + architecture × 0.20
  + devops       × 0.15
  + dependency   × 0.10
  + governance   × 0.05
```

Total weight = 1.00. **Every scoring skill is now included** — the earlier formula (which summed to 1.00 across only 5 skills) is deprecated.

`claude-metrics` is a **meta-skill**; it records telemetry (tokens, duration, tool calls) and produces **no score**.

### Missing-skill behaviour

If a skill is skipped (disabled in config, failed to run, or not applicable) its dimension score is `null`. The overall score is recomputed over the remaining weights, renormalized to 1.00:

```
overall = Σ (score_i × weight_i) / Σ (weight_i)  over skills with score ≠ null
```

Skipped skills are listed in `runs[].properties["repo-intel.skippedSkills"]` with a reason string.

---

## 3. Required outputs per skill

Every scoring skill MUST return:

1. A numeric `score` in `[0.0, 10.0]`.
2. A **score factor table** explaining how the score was derived. Columns (exact, in order):

   | Factor | Rule ID | Severity | Count | Base impact | Total deduction |
   |--------|---------|----------|------:|------------:|----------------:|

   One row per rule that fired. Sum of `Total deduction` plus `score` must equal 10.0 (modulo caps).
3. A `findings[]` list conforming to `skills/FINDING-SCHEMA.md`.
4. Optional: `metrics{}` dictionary for dimension-specific measurements (lines of code, coverage estimate, CVE count, etc.).

See `skills/SUBAGENT-OUTPUT.md` for the exact wire format.

---

## 4. Reproducibility rule

A score is reproducible if, given the same `findings[]` list, the score factor table, and this contract, an independent reviewer can derive the same score within ±0.1. If not, the skill is buggy, not the contract.

---

## 5. Changing the contract

This file is versioned together with `output/report.schema.json`. A change to severity multipliers or dimension weights is a **major** version bump (`repo-intel.schemaVersion`) and requires updating every skill, the example report, and the CHANGELOG.
