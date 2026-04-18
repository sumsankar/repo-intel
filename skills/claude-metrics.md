# Skill: claude-metrics

Capture and report the token usage, context window utilization, and memory consumption for the current Claude Code analysis run.

---

## When to run

Run this skill **last**, immediately after the `governance` skill completes and before writing the report.

---

## What to collect

### 1. Timing

Record the wall-clock start time at the beginning of Step 3 (before the first skill runs) and the end time when this skill executes. Compute:

```
duration_seconds = end_time - start_time
```

### 2. Token usage (estimate)

Claude Code does not expose raw token counts via the shell, so derive them from file sizes and conversation content:

```bash
# Total bytes read from the target repo during analysis
du -sb /tmp/repo-intel-target 2>/dev/null | awk '{print $1}'

# Count files opened (Read tool calls)
# Count bash commands executed (Bash tool calls)
```

Use the following conversion factors for estimates:

| Source                                         | Tokens                   |
| ---------------------------------------------- | ------------------------ |
| 1 KB of source/markdown content                | ~250 tokens              |
| 1 KB of bash command output                    | ~250 tokens              |
| Each user message                              | ~50–200 tokens           |
| System prompt (ANALYZE.md + all skills loaded) | count the characters ÷ 4 |

Compute:

- **Input tokens** = tokens from user message + tokens from all skill files loaded + tokens from all file/command output read during analysis
- **Output tokens** = tokens in all report content written (estimate: report file size ÷ 4)
- **Total tokens** = input + output

### 3. Context window utilization

Claude models have a **200,000-token** context window.

```
context_utilization_pct = round((total_tokens / 200000) * 100, 1)
```

Flag the utilization level:

- < 25% → 🟢 Low
- 25–60% → 🟡 Moderate
- 60–85% → 🟠 High
- > 85% → 🔴 Near limit

**Target budget (optimized):** < 120K input + < 15K output = < 135K total (< 68% utilization)

### 4. Memory files loaded

Check whether Claude Code loaded any memory files from `.claude/` at session start:

```bash
ls -la .claude/ 2>/dev/null
ls -la ~/.claude/projects/ 2>/dev/null | head -5
```

List each file loaded and its size in KB.

### 5. Tool call tally

Count the number of tool uses made during the analysis run across these categories:

| Tool              | Count |
| ----------------- | ----- |
| Bash              |       |
| Read              |       |
| Write             |       |
| Edit              |       |
| Grep              |       |
| Glob              |       |
| Agent (subagents) |       |
| **Total**         |       |

### 6. Files accessed in target repo

```bash
# Estimate number of files inspected
find /tmp/repo-intel-target -type f | wc -l
```

---

## Output format

Produce a `ClaudeMetricsResult` record with these fields:

```
model:                  claude-sonnet-4-6
run_date:               <ISO 8601 date>
run_duration_seconds:   <integer>
input_tokens_est:       <integer>
output_tokens_est:      <integer>
total_tokens_est:       <integer>
context_window:         200000
context_utilization_pct:<float>
context_pressure:       🟢 Low | 🟡 Moderate | 🟠 High | 🔴 Near limit
memory_files_loaded:    <list of file paths and sizes, or "none">
tool_calls_total:       <integer>
tool_calls_breakdown:   <dict>
repo_files_inspected:   <integer>
```

---

## Population instructions for the report

Fill the `<!-- Claude Run Metrics -->` section in both `report-template.html` and `report-template.md` with the values above.
Be transparent that token counts are **estimates** derived from file sizes and conversation content — not values read from an API response.

---

## Skill classification

`claude-metrics` is a **meta-skill**. It produces no score, no findings, and no `RI-*` rule IDs. It contributes to the SARIF document as follows:

- `runs[].invocations[0].properties`:
  - `model` — e.g. `claude-opus-4-7`
  - `tokens_input_estimate` — integer
  - `tokens_output_estimate` — integer
  - `context_utilization_pct` — float 0–100
  - `tool_calls` — integer
- `runs[].properties["repo-intel.skippedSkills"]` — any skill that was disabled or failed.

### Token estimation table

Use per-language factors instead of a flat `1 KB = 250 tokens`:

| Content type | Tokens per KB |
|--------------|--------------:|
| TypeScript / JavaScript | 250 |
| Python / Ruby / PHP | 200 |
| Go / Rust / C# / Java | 220 |
| YAML / TOML / INI | 180 |
| JSON | 150 |
| Markdown | 200 |
| HTML | 100 |
| CSS | 180 |

**Return format:** return a [SUBAGENT-OUTPUT.md](SUBAGENT-OUTPUT.md) envelope with `status: ok`, no `score`, no `findings`, and the metrics block above under `metrics:`.
